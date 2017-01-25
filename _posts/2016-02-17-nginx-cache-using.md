---
layout: post
title: "nginx cache using"
description: ""
category: [nginx]
tags: []
---
## nginx缓存的使用
建立缓存系统是优化各种应用性能的常见手段，本文介绍nginx对被代理的后端服务所提供的缓存功能，即web content caches。这种缓存有两个好处：1）提升响应给客户端内容的性能，2）减少后端服务器的负载。这层的缓存策略需要视访问的内容而定，比如对于一些图片、js、css等静态内容，可以缓存相对长时间；对于一些完全私有、易变的内容，缓存有时不是一个好主意；还有一些非私有、但是动态生成、不定期变化的内容，缓存策略的选取就是个需要考虑的问题，这就是所谓的microcaching。

<!--more-->

microcaching是指内容仅缓存一个非常短的时间，可能就1s，这意味着网站对用户的数据延迟不超过1s，通常这是完全可接受的。

[文章](https://www.nginx.com/blog/benefits-of-microcaching-nginx/)给出了一个使用nginx缓存的实例，一步步优化一个基于wordpress搭建的blog应用，文章中通过增加nginx缓存将原来qps不到6硬是提升到逼近3000，效果拔群。

下面就来介绍下nginx缓存的使用和调优，主要来自[文章](https://www.nginx.com/blog/nginx-caching-guide/)

### 配置基本的caching功能
主要涉及两个命令：proxy_cache_path和proxy_cache。proxy_cache_path设定cache文件路径以及配置cache，proxy_cache激活它。

{% highlight c %}
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m 
                 use_temp_path=off;

server {
...
    location / {
        proxy_cache my_cache;
        proxy_pass http://my_upstream;
    }
}
{% endhighlight %}

下面分别解释下proxy_cache_path中各个参数的含义：

- /path/to/cache 指定的是装载了的本地disk目录。
- levels设置了一个在/path/to/cache中的2级目录结构用来存放缓存content文件。
- keys_zone设置了一个shared-memory zone，用于存放cache keys和一些metadata data（比如：usage timers）,将keys存放在这个shared-memory区域中使得所有worker进程能够快速知道某个key是否存在，1MB能存放8K个keys。
- max_size设置cache内容总大小的上限
- inactive指定某个item能被保存在cache中多久而没有被访问过。在本例中是60min，意味着如果某个文件在60min内没有被访问过，那么nginx的cache manager进程就会将它从cahce中删除。该值默认是10min。注意该值和expired content的区别，比如后端服务器的响应中指定了cache-control:max-age=120，意味着2min后内容就expire了，nginx不会自动删除expired的内容，依然存在于缓存中，但当客户端再次请求该expired内容时，nginx会refreshes it from the origin server and resets thr inactive timer。
- nginx会将文件写到一个临时存储区域，然后rename到/path/to/cache中相应目录下，use_temp_path指定这个临时的存储区域，当值为off时表示直接写到目标cache目录，以避免文件在不同文件系统间拷贝

### 不新鲜的内容也好过什么也没有
nginx 内容缓存一个强大的特性就是可以配置让nginx当没有从后端及时取到新内容时响应客户端以一个存在于cache中的stale内容（expired content），没有从后端取到新内容的原因可能是后端服务down掉或者正忙，此时响应一个旧的存在于cache中内容, 要好于对客户端响应一个错误

{% highlight c %}
location / {
    ...
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
}
{% endhighlight %}

上述配置中指示如果当nginx接受到error、timeout、或者5XX错误，同时缓存中存在该请求旧的响应内容，那么该旧的内容会被响应给客户端。

### Fine-Tuning the cache
nginx提供了一些tuning的指令，举例说明：

{% highlight c %}
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m
                 use_temp_path=off;

server {
    ...
    location / {
        proxy_cache my_cache;
        proxy_cache_revalidate on;
        proxy_cache_min_uses 3;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_cache_valid 200 1s;
        proxy_pass http://my_upstream;
    }
}
{% endhighlight %}

- proxy_cache_revalidate指示nginx当需要去后端服务器刷新内容时使用conditional Get请求。比如后端服务器的指定了内容的过期时间，当客户端发请求时内容已经过期，那么nginx需要重新去向后端服务器请求内容，但是proxy_cache_revalidate为on时，nginx请求将包含If-Modified-Since头(值为内容的Last-Modified)，后端服务器只有当自该时间点后修改过才会去响应内容，否则304，这样节省带宽。
- proxy_cache_valid，这个指令可以为不同的响应code指定不同的在nginx端的缓存时间，例子中，为200的响应指定了缓存时间为1s，意味着1s后数据就过期了(expired)，但注意不是1s后自动删除了啊。当客户端再次请求该过期内容时，nginx就得去后端服务器去取了。

- proxy_cache_min_uses设置只有经过几次访问，内容才会被nginx缓存，默认为1。这个设置主要是针对那些缓存经常被打满了场景，它保证只会缓存那些经常被访问的内容
- proxy_cache_use_stale前面提到了，指定使用旧数据的场景，多了个updating，它告诉nginx，当同一个过期的内容有多个请求访问时，第一个请求将去后端服务器访问最新的数据，其他后续请求先使用旧数据，第一个请求则等待响应结束。

- proxy_cache_lock是用于当多个请求访问同时同一个未缓存的内容时（MISS），缓存被击穿，为了防止流量导致后端服务器过载，nginx仅允许第一个请求向后端服务器访问数据，其他的请求先等待直到nginx将第一个请求的响应缓存。

### 缓存可以位于不同的hard drives上
{% highlight c %}
proxy_cache_path /path/to/hdd1 levels=1:2 keys_zone=my_cache_hdd1:10m max_size=10g 
                 inactive=60m use_temp_path=off;
proxy_cache_path /path/to/hdd2 levels=1:2 keys_zone=my_cache_hdd2:10m max_size=10g 
                 inactive=60m use_temp_path=off;

split_clients $request_uri $my_cache {
              50%          “my_cache_hdd1”;
              50%          “my_cache_hdd2”;
}

server {
    ...
    location / {
        proxy_cache $my_cache;
        proxy_pass http://my_upstream;
    }
}
{% endhighlight %}

上面的例子中指定了两个cache,(my_cache_hdd1 and my_cache_hdd2)位于不同的hard drive上，split_clients指令配置将请求映射到两个cache的方法。

### 一些杂项

**nginx如何决定内容缓存或者不缓存**: 

默认情况下，即便是配置了proxy_cache，nginx也不是对所有响应都进行缓存，nginx遵从后端服务器的Cache-Control，对于Cache-Control为private, no-cache, no-store或者响应头中有set-cookie的都不会进行缓存。另外，默认情况下，nginx只会去缓存客户端的GET和HEAD请求。当然可以配置让nginx覆盖这些默认行为，比如proxy_ignore_headers Cache-Control; proxy_cache_methods GET HEAD POST; proxy_cache_bypass

**nginx缓存使用什么key**:

默认情况下，是$scheme$proxy_host$request_uri，比如对于配置：
{% highlight c %}
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m
                 use_temp_path=off;

server {
    ...
    location / {
        proxy_cache $my_cache;
        proxy_pass http://my_upstream;
    }
}
{% endhighlight %}

请求 http://www.example.org/my_image.jpg 的key是md5("http://my_upstream:80/my_image.jpg")，注意$proxy_host是proxy_pass指定的。

可以用proxy_cache_key指令覆盖默认的行为，比如：proxy_cache_key $proxy_host$request_uri$cookie_jessionid;

