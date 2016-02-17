---
layout: post
title: "nginx basic"
description: ""
category: 
tags: []
---
##Nginx Basic
本文主要是对nginx的配置做了一些简要的介绍，来源是nginx的官方文档，结合自己的理解做了下精简和翻译，一来，为了加深对各个配置的理解，二来，为下一步读nginx的源码做准备

### location的概念 
- nginx配置文件中支持配置多个server，监听不同端口(也可以监听同一个端口，如果没有listen，默认是80端口。另外一个server还可以监听多个端口)。
nginx的server是一个非常松散、灵活的概念

<!--more-->

- location命令是用于指定一个前缀(或者正则表达式)去匹配uri，然后映射到块内root（或者全局root）指定的目录当中(或者转发给快内proxy_pass指定被代理服务器，或者转发给快内fastcgi_pass配置的FastCGI server)。
- 如
```
location /blog {
    root /site;
}
```
意味将url前缀为/blog的请求访问的文件都从/site目录里去找，/blog/post.html则实际上访问的是/site/blog/post.html
- 一个server可能有多个location，以匹配不同的url。当一个请求过来后，对请求的url，nginx的处理流程是先检查前缀匹配的locations, 记住能匹配到最长前缀的location，再检查正则匹配的location，如果匹配到了正则表达式的location，则选择这个location，否则，选择之前记住的最长前缀的location

### server匹配
一个server可以配置一个server_name用于匹配request的Host。另外，一个nginx实例中可以配置多个server，意味着机器上可以配置多个站点，（注意站点和端口没有必然的一对一关系，可以多个server公用一个端口）。那么问题来了，一个请求过来了匹配到具体server的规则是什么：
 * 首先肯定得按照请求的IP和port匹配listen命令，前面说到，多个server可能有相同的listen，进入下一条
 * 最直观的就是按照请求的host匹配server_name, server_name可以指定为多种形式
 * 如果没有server_name匹配到host，那么请求会被该ip该端口上的默认server处理（default_server）
 
### server_name匹配基本规则
前面说到，server_name可以有多种形式：（实际名称、通配符、正则表达式）
{% highlight c %}
server {
    listen       80;
    server_name  example.org  www.example.org;
    ...
}

server {
    listen       80;
    server_name  *.example.org;
    ...
}

server {
    listen       80;
    server_name  mail.*;
    ...
}

server {
    listen       80;
    server_name  ~^(?<user>.+)\.example\.net$;
    ...
}
{% endhighlight %}
如果当请求的host匹配到多个server_name，那么最终选择server_name的规则是啥? 会按照如下优先级去选择：
1. 实际准确的名称
2. 以*开始的最长通配符名称 比如：*.example.org
3. 以*结尾的最长通配符，比如：mail.*
4. 最后选择第一个匹配的正则表达式server_name(nginx的正则语法和perl的语法一致，需以~开始，正则写在^和$之间)


### server_name匹配杂项规则
1. server_name为"",可以匹配那些没有host feild的请求
2. server_name可以指定为$hostname, 匹配host为访问机器的hostname的请求
3. server_name可以指定为_，可以匹配所有的host，一般来说，_作为任何正常host都不会是的server_name，存在的意义在于，可以设定为default_server, 然后return一个自定义的错误(444)

### 优化
实际名称、*开始、*结束通配符server_name会被存在绑定在监听端口的三个hash tables中，哈希表的size在配置阶段就被最优化了，以便以最少的cpu cahce miss去匹配name。这个hash表是用于找到对应的server的。

实际名称的hash表查的速度最快，通配符次之。正则没有hash表，而是一个一个地匹配的，因此不能大量去应用。建议是将一些被请求频率最高的host，设置为实际名称的server_name。

如果很多定义了很多server_name， 或者有些很长的server_name，那么server_names_hash_max_size和server_names_hash_bucket_size可能需要指定下


## 将nginx作为负载均衡器
nginx支持三种负载均衡机制：

- round-robin，没啥好说的
- least-connected，least_conn指令, nginx根据每台后端机器的连接数来选择把请求发送那台去
- ip-hash，当需要将一个client的请求始终发送同一台后台机器以保持会话时使用。

一个负载均衡配置的实例如下：
{% highlight c %}
http {
    upstream myapp1 {
        //round-robin
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
{% endhighlight %}
上面没有指定负载均衡的机制，默认是round-robin。所有的80端口上请求都被代理到myapp1集群上去，该例子代理的是http请求，如果后端支持https，直接替换成https即可，如果后端是fastCGI\uwsgi\SCGI\memcached，那么则需要fastcgi_pass/uwsgi_pass/scgi_pass/memcached_pas等指令了。

负责均衡除了上述几种机制外，还可以结合加权机制。在upstream中的server后面加上weight。

nginx还支持health check，主要通过两个指令，fail_timeout 和max_fails。fail_timeout指定再次探测服务是否可用的时间间隔，max_fails指定连续探测几次后认定后端服务不可用，默认为1，为0时表示禁用health check功能。那么什么才是一次后端服务不可用的认定呢？是由根据后端服务的类型而独立定义的（proxy_next_upstream, fastcgi_next_upstream, uwsgi_next_upstream, scgi_next_upstream, memcached_next_upstream)，它们指定了什么情景下请求将被转发到下一个后端机器上。比如proxy_next_upstream的使用如下：
{% highlight c %}
Syntax: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | off ...;
Default: proxy_next_upstream error timeout;
Context: http, server, location
{% endhighlight %}

* 注意上述的health-check类型是in-band health-check，意味着不主动去check，而是随着请求来的。nginx plus支持out-of-band的health-check，持续地check upstream中的服务可用性。

* 还有一点需要注意，指令都有上下文，在使用的时候记得查下文档看下指定的适用上下文。

* 上面介绍的只是配置nginx负载均衡的一些基本指令，更全面的的文档见[ngx_http_upstream_module](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)

## HTTPS server配置
首先得在server block里配置的listen指令里加上ssl参数，表示开始https认证

指定https的认证的参数（证书、private key file），证书是个public的实体，将会被发送给每个访问server的client。private key file是个secure entity，应该被限制访问，nginx的master进程应该具有读权限。（这个private key file到底在TLS协议中是什么）

HTTPS认证需要额外的计算量，减少计算量的方法有两种：
 - 开启keepalive，复用已经建立的连接
 - 复用SSL session, 以避免SSL握手的开销（证书不需要重新验证、对称加密的密钥不需要重新重新协商）。SSL sessions被存储在由多个workers共享的session cache中（由ssl_session_cache指令配置cache的大小，ssl_session_timeout配置cache中session的过期时间）
 
## 谈谈nginx的整体架构
本节的内容主要翻译自[nginx](http://www.aosabook.org/en/nginx.html)

先放一张nginx整体的架构示意图
![architecture](/public/fig/architecture.png) 

图中可以看出：worker直接负责处理client的请求。然而，nginx中没有一个专门来分发连接到worker的机构，这是由OS内核的机制决定的。一旦nginx启动，一个listening socket集合就被创建了，worker就开始各自accpet连接，read/write socket。那么问题来了:  **怎么对worker去分配listenging socket或者说listen port。**

### worker数量的选择
worker数量应该随着disk的使用和cpu的负载模式不同而有所调整。基本规则是，对于对于CPU密集的服务比如处理大量TCP/IP、 处理SSL、压缩等任务时，worker数量应该等于CPU核心数；对于瓶颈是I/O的服务，比如负载很重的代理、需要从disk加载不同的content的服务，worker的数量最好是CPU核心数的1.5-2倍。

在后续的nginx版本中，开发者将逐渐解决针对disk的block访问（EPOLL貌似不支持disk i/o）。就目前而言，如果没有足够的存储性能时worker的disk访问将会被阻塞。当然目前存在一些机制和指令去缓和这一情况比如：sendfile和AIO。

另一个nginx的问题是对于嵌入式脚本的支持有限，标准的nginx版本中，仅支持Perl脚本，对此也有个简单的解释：关键是嵌入式脚本可能会阻塞或者异常退出，这会导致worker被hang住。

### nginx进程角色
nginx在内存在运行一个master进程，多个worker进程，还有一些特殊目的的进程（一个cache loader，一个cache manager）。所有进程都是单线程的，进程间主要使用shared-memory机制进行IPC。master进程以root用户运行。其他以unprivileged user运行。

master进程负责：

- 读取和验证配置文件
- create \ bind \ close socket
- 启动、终止、维护预先配置的数量的worker
- reconfiguring without service interruption
- controlling non-stop binary upgrade
- re-opening log files
- 编译嵌入式Perl脚本

worker进程负责：

- 接受、处理客户端的连接
- 提供反向代理和过滤器功能
- 其他nginx提供的功能

cache loader进程负责：checking the on-disk cache items and populating nginx's in-memory database with cache metadata.  Essentially, the cache loader prepares nginx instances to work with files already stored on disk in a specially allocated directory structure. It traverses the directories, checks cache content metadata, updates the relevant entries in shared memory and then exits when everything is clean and ready for use.

 cache manager进程负责：cache过期校验。
 
### nginx缓存系统概述
 nginx缓存的实现是一种基于文件系统的层次数据存储形式。具体使用见[nginx官方文档的ngx_http_proxy模块](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)。还有一篇专门介绍nginx反向代理cache使用的[blog](https://www.nginx.com/blog/nginx-caching-guide/)。
