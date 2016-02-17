---
layout: post
title: "nginx cache using"
description: ""
category: 
tags: []
---
## nginx缓存的使用
建立缓存系统是优化各种应用性能的常见手段，本文介绍nginx对被代理的后端服务所提供的缓存功能，即web content caches。这种缓存有两个好处：1）提升响应给客户端内容的性能，2）减少后端服务器的负载。这层的缓存策略需要视访问的内容而定，比如对于一些图片、js、css等静态内容，可以缓存相对长时间；对于一些完全私有、易变的内容，缓存有时不是一个好主意；还有一些非私有、但是动态生成、不定期变化的内容，缓存策略的选取就是个需要考虑的问题，这就是所谓的microcaching。

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

- proxy_cache_revalidate指示nginx使用conditional Get去

