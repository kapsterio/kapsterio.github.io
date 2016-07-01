---
layout: post
title: "about netty"
description: ""
category: 
tags: []
---
# 关于netty
好几月没写博客了，水篇杂文把，写点最近学到的：

## attributeMap的使用
ChannelHandlerContext和Channel都继承了AttributeMap，也就是说他们都可以充当attributeMap。一般来说，一个channel就是一个链路（底层对应一个socket），它有一个业务handlers组成的pipeline，通常我们希望每个handler都是stateless的，这样handler对象就可以shared by everyone，然而有时handler有需要保存state的需求，那么此时可以利用pipeline上handler对象关联的context对象（也就是说每条pipeline上有这与handler数量相对应的context对象，在handler的override方法里都有context参数）来保存attribute。Channel对象的attribute有什么用呢，通常我们会把Channel对象保存在一个应用程序全局的地方，channel里的attribute就为我们提供了一种优雅实现保存连接属性的方法（比如之前的http的keepalive就可以放在着）。更多内容参见官方文档中[state management](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html)，以及[channelHandlerContext](http://netty.io/4.0/api/io/netty/channel/ChannelHandlerContext.html)

<!--more-->

## 关于基于netty写一个proxy的例子
我之前的理解是有问题的，即一个handler即作为服务端也作为客户端时如何共用eventloop，之前不知道怎么理解成了：共用eventloop后，pipeline也相同了。当然，作为服务端和客户端时底层连接都不一样，不是同一个channel，pipeline自然也不一样。他们只是共享eventloop而已（即在同一个selector上注册，由同一个线程处理）。

那么就有个问题，每次前端请求来（即一个前端channel建立）是否都要去建立一个到后端的channel，例子程序中是这样做的，显然不大合适，理想的做法是应用启动时建立一个到后端的连接的channel pool，把这些channel加入与前端连接相同eventloop中。当需要用到后端channel时去pool中借出一个channel，再向这个channel非阻塞地write请求，不等待响应就去处理其他前端请求了。

问题就变成了当后端channel响应后（netty可能会调用channel上pipeline中handler的channelRead0方法），怎么去找到使用它的前端channel呢？一个naive的想法就是在后端channel中存下前端channel的引用，比如可以把前端channel存入后端channel的attributemap中，什么时候存入呢，就在从pool中借出后端channel时存入。这样一来当后端channel请求响应了后，从其attribute中再取出前端channel将响应msg写给前端channel，再把前端channel的引用从后端channel的attr里删除，再把后端channel还给pool。

这时就产生了另外的问题，如果后端channel响应时间过长怎么办，我们应该为后端channel设置一个响应时间，超过这一时间了也没见着后端channel的响应，我们就要给前端channel响应一个超时msg。怎么实现呢？带着这个问题，我回顾了下ev_lib中timer的做法，它基于最小堆和Linux的timefd实现，timefd和I/O fd一起被epoll；看了下java timer的实现，它基于最小堆和内部条件队列实现，单线程执行定时任务，excecutor框架中的ScheduledExecutorService实现机制类似，只是内部采用多个线程去执行任务；了解了下netty的定时任务实现，基于所谓的Hashed  and  hierarchical  timing wheels。

那么基于netty，上面的问题就可以这样实现：当发起一个后端channel的write后就向当前eventLoop中schedule一个定时任务，得到一个Future，时间就设置为超时时间，任务内容是向前端channel响应一个超时msg，并关闭后端channel（谁让你超时）。如果后端channel及时响应了，需要将代表这个task的future取消掉（cancel）。因此后端channel的attr里还需要保存一个future的引用。

ok，实现思路就是这样，回头研究下netty关于定时任务的详细实现。

对于这个后端channel pool，我们需要在应用里定时去发送心跳，如果channel没有响应，我们还需要将其从pool里删除。同上，这部分也可以基于netty的定时任务来实现。顺便说下，在 netty in action这本书的8.3节提到了两个channelhandler——IdleStateHandler和ReadTimeoutHandler，感觉刚好简化了上述需求的处理。其中,IdleStateHandler比较好用，可以在建立后端channel的pipeline时加上，ReadTimeoutHandler不怎么适合需求，我的需求是发起write后的某个时间内没有响应就认为超时，然而ReadTimeoutHandler貌似是只有在固定时间内没有响应就认为channel超时，从而关掉连接。

最后一个问题，关于channel pool，了解了下apache common pool的使用和实现，非常不错的通用pool，极大地简化了pool的开发，这篇[blog](http://shift-alt-ctrl.iteye.com/blog/1917782)写得很赞，也有实现骨架的源码。我只要去重载PoolableObjectFactory的几个方法就ok了。apache大法好。

## ForkJoin framework
就是分而治之思想的高效并行化实现，不同于executors的地方在于每个work线程都有一个工作队列，而且这个队列大部分情况下是lock free的，另外当一个worker没有任务可做时它还可以去别的work queue中偷任务，如果没有偷到可能就把自己的运行优先级调低，或者让步，再或者睡眠。如果所有线程都没有任务可做，则block。一篇[oracle关于forkjoin的文章](http://www.oracle.com/technetwork/articles/java/fork-join-422606.html)很好介绍了它的使用、优点，并且举了非常赞的文件夹中并行查找单词的例子。[这篇文章](http://homes.cs.washington.edu/~djg/teachingMaterials/grossmanSPAC_forkJoinFramework.html)也有着很好使用方法的介绍。猎豹移动的博客上有篇[关于forkjoin源码分析](http://dev.cmcm.com/archives/87)，至少比并发编程网上写的要靠谱些。

## netty5.0
其中中值得注意的一些变动: [noteworthy in 5.0](http://netty.io/wiki/new-and-noteworthy-in-5.0.html)

水完了，这篇文章压根就不能看。。

