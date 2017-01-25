---
layout: post
title: "cURL in PHP and HttpClient in Java"
description: ""
category: [httpclient]
tags: []
---
## 背景
我在公司所负责的抵用券模块对公司内其他业务方提供一些抵用券相关的http接口（php实现），另外对外界用户客户端提供一些查询抵用券的thrift接口（java实现），现状是后者会请求前者去获取所有抵用券的数据。公司经过组织结构调整后，各业务方也相继实现了自己专用抵用券系统，然而各业务方都希望能在原有的客户端抵用券入口处展示自己的数据，那么问题来了：在不变动客户端原有的获取抵用券数据逻辑下，只能是由我们在接口里做数据的聚合工作。

<!--more-->

聚合就要涉及到请求多个不同业务方的获取数据的接口，显然，以同步的、级联的方式去请求各个接口是不合适的，不仅会导致我们的接口响应时间增大，而且任何一方接口的不稳定都会影响我们接口的稳定性，这个风险随着需要请求的业务方接口数量的增加而增大。首先想到的解决方案是通过开多个线程同步且阻塞地去请求各接口，等所有线程取到数据后再做数据聚合；或者基于操作系统提供的I/O多路复用以非阻塞的方式在单线程里发起对多个接口的请求，然后等有FD可读了就取到数据，直到取完所有数据。

## multi exec in cURL[kә:l]
因为我们组在生产环境下的主力语言是php，所以首先尝试在php里去干这事（其实对于php这种请求处理完了就释放所有资源的语言，是不适合做这件事的，后面会提到）。php里发起http请求最常用的工具就是cURL（来，脑补一下发音：克儿），cURL是基于c实现的libcurl的。自然地首先看看cURL有没有封装相关的函数供使用，然后就找到了这个函数：**`curl_multi_exec`** 和 **`curl_multi_select`**。

**`curl_multi_exec`**作用是发起多个http请求，按照libcurl文档上的描述，它的作用之一就是往多个socket FD里以非阻塞的方式write进http请求数据，并不等待请求的响应。**`curl_multi_select`**的作用看着名字也可以知道，其实就是select，即让进程wait直到有socket FD的I/O事件发生。

php manual上还给了个example，这里贴一下：

{% highlight php linenos %}
<?php
// create both cURL resources
$ch1 = curl_init();
$ch2 = curl_init();

// set URL and other appropriate options
curl_setopt($ch1, CURLOPT_URL, "http://lxr.php.net/");
curl_setopt($ch1, CURLOPT_HEADER, 0);
curl_setopt($ch2, CURLOPT_URL, "http://www.php.net/");
curl_setopt($ch2, CURLOPT_HEADER, 0);

//create the multiple cURL handle
$mh = curl_multi_init();

//add the two handles
curl_multi_add_handle($mh,$ch1);
curl_multi_add_handle($mh,$ch2);

$active = null;
//execute the handles
do {
    $mrc = curl_multi_exec($mh, $active);
} while ($mrc == CURLM_CALL_MULTI_PERFORM);

while ($active && $mrc == CURLM_OK) {
    if (curl_multi_select($mh) != -1) {
        do {
            $mrc = curl_multi_exec($mh, $active);
        } while ($mrc == CURLM_CALL_MULTI_PERFORM);
    }
}

//close the handles
curl_multi_remove_handle($mh, $ch1);
curl_multi_remove_handle($mh, $ch2);
curl_multi_close($mh);

?>
{% endhighlight %}

这里解释下这段看起比较诡异的代码。

#### 问题一：
为什么对curl_multi_exec的调用首先要包在一个do while循环里，这段是干嘛的？前面说到curl_multi_exec是以non-blocking的方式往socket fd里写数据，也就意味着底层的write可能立即返回个EAGAIN什么的，那么此时就要重新去write，意味着要再次调用curl_multi_exec，libcurl官方文档上对curl_multi_perform（curl_multi_exec的底层实现）有着这样一段描述：

> curl_multi_perform(3) is asynchronous. It will only execute as little as possible and then return back control to your program. It is designed to never block. If it returns CURLM_CALL_MULTI_PERFORM you better call it again soon, as that is a signal that it still has local data to send or remote data to receive.

##### 问题二：
第二个while循环是干嘛的？前面的第一个do while循环已经将请求数据都write过去了，现在要干的事就是等有响应数据过来。其中active大于0时表示还存在socket的响应数据没有过来。为了不让cpu以100%占用率去轮询数据有没有过来，一个合理的方式就是数据过来后，操作系统告知应用进程，这也就是curl_multi_select的作用，它使得应用进程先去wait，等有数据过来时再wake up。

#### 问题三:
第二个while循环里面的第三个do while是干嘛的？前面说到，当有数据来时，curl_multi_select将应用进程唤醒，醒来后要去相应的socket上去处理数据，此时就需要再次用到curl_multi_exec，基于之前同样的原因（non-blocking）对curl_multi_exec的调用需要包在这个do while循环里。

ok，代码解释完了，需要说明的是，**由于没有去读cURL和libcurl的源码，上述内容只是我的合(hu)理(luan)推(cai)测(ce)**。

等等，好像还少了什么，我要从各个http请求里取到响应数据啊。这个也容易，正常情况下，第二个大while循环结束后，意味着active已经为0了，即所有http请求的响应数据也都有了，那么对每个handle调用一次curl_multi_getcontent，这个函数返回一个string，php代码里就得到了响应数据。

#### 总结与讨论：

- 假定各个接口成功返回全部数据后，应用逻辑里处理这些数据的延迟很小(一般情况下是这样的，数据都在内存里)。那么这段代码的处理时间就取决于所有接口中响应时间最长的那个接口的响应时间。所以我认为[Rolling cURL: PHP并发最佳实践](http://www.searchtb.com/2012/06/rolling-curl-best-practices.html)这篇文章中的做法也就没什么必要了。

- www代码库中的CUrlHttp.php提供了httpMultiGet函数，它每调curl_multi_exec一次，然后usleep(10000)的做法有点偷懒，为什么不去用curl_multi_select呢?

- 在业务逻辑里，往往需要访问的几个后端接口是固定的，在一个并发较高的逻辑里，最理想的方式是为每个需要访问的后端接口维护一个连接池，这样就省去了每次请求后端接口都需要建立连接和关闭连接的时间（tcp三次握手和四次挥手），然而，对于php而言，这是做不到的。。（或者说我不知道怎么才能做到）

## apache HttpClient
前面说到php里基于I/O多路复用地异步请求多个接口来做数据聚合并不是最合适的方法，而且公司内部好像也没踩过这方面的坑，遂没在生产环境下使用上面的解决方案。因此准备将该工作放到提供给客户端调用的java实现的thrift接口里。java应用运行在java容器里面，可以实现分配一定数量的连接池资源用于服务客户端来的并发请求，请求处理完了后，资源也不需要释放，可以很好的复用，看上去非常适合做这个事情。

首先调研下了公司有没有现成的封装好了的方法可以供使用，得到的结论是没有。。公司的fundmental库只提供了一个基于apache HttpClient的、同步的、单线程的访问http接口的工具HttpClientHelper。因此需要自己实现。

这就需要先了解下apache的httpComponents组件了，httpComponents是封装http以及相关协议的一些工具集合，由三个大组件构成httpCore、httpClient和httpAsyncClient。这里直接引述[apache基金会官方文档](https://hc.apache.org/index.html)对它们的介绍：

> HttpCore is a set of low level HTTP transport components that can be used to build custom client and server side HTTP services with a minimal footprint. HttpCore supports two I/O models: blocking I/O model based on the classic Java I/O and non-blocking, event driven I/O model based on Java NIO.

> HttpClient is a HTTP/1.1 compliant HTTP agent implementation based on HttpCore. It also provides reusable components for client-side authentication, HTTP state management, and HTTP connection management. 

> Asynch HttpClient is a HTTP/1.1 compliant HTTP agent implementation based on HttpCore NIO and HttpClient components. It is a complementary module to Apache HttpClient intended for special cases where ability to handle a great number of concurrent connections is more important than performance in terms of a raw data throughput.

可以看出，HttpClient是基于blocking I/O model，而HttpAsyncClient是基于java NIO。apache官方文档上关于HttpClient提供了丰富的例程和翔实的Tutorial，关于HttpAsyncClient只提供了一些例程。我们的线上thrift接口分摊到每台机器上的qps约为20，需要同时请求3个后端接口，因此同时访问的socket FD个数约为60个。我在线下实验中模拟这个并发量下，基于HttpClient的多线程同步请求和基于HttpAsyncClient异步请求后端接口性能差别不大（都是比响应时间最长的那个后端接口响应时间多一点点）。基于上述原因，我采用多线程的HttpClient实现并发访问多个后端接口。

## HttpClient的使用

#### 初始化：
{% highlight java linenos %}
RequestConfig config = RequestConfig
            .custom()
            .setConnectTimeout(100) //三次握手时建立连接的超时
            .setConnectionRequestTimeout(100) //从connectManager中获取connection的超时
            .setSocketTimeout(500) //从socket中读数据的超时
            .build();
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(100);
cm.setDefaultMaxPerRoute(20);
CloseableHttpClient client = HttpClients.custom()
            .setUserAgent("MOBILE_SERVER")
            .setDefaultRequestConfig(config)
            .setConnectionManager(cm)
            .build();
{% endhighlight %}

这是一段相对较完善的构造出一个HttpClient对象的代码片段，一般来说，一个应用中只需要一个HttpClient实例，通常可以用单例模式来实现。这段代码的主要工作就是为HttpClient配置一些参数以及连接管理器(HttpClientConnectionManager)，简称CM。

我们知道自HTTP1.1开始，默认是基于长连接的，意味对某个host的多个http请求可以复用同一个底层的TCP连接，节省连接建立和释放的开销。HttpClient也是支持长连接的，它由CM来管理底层的连接，有两种CM:BasicHttpClientConnectionManager（BCM）和PoolingHttpClientConnectionManager（PCM）。

BCM非常简单，在底层只维持一个connection对象，也是支持长连接的，如果后续请求的是同一个host，该connection对象可以被复用，如果后续请求其他host，那么BCM会先重开一个新的connection用于服务。注意一点：**虽然BCM是线程安全的，但应该只被用于单线程环境下，因为它底层只有一个connection，多线程使用时势必要加锁，性能会有问题**。

PCM相对复杂，它为每个host管理一个连接池，可被用于多线程环境下中。每个线程向连接池去租用连接，用完了再还给连接池。我们可以配置这个连接池的大小以及总连接池的大小，即代码中的cm.setMaxTotal(100)和cm.setDefaultMaxPerRoute(20)。默认情况下这两个值分别是20和2，一般来说都是不够用的。

#### 关闭和释放资源
{% highlight java linenos %}
CloseableHttpClient httpClient = <...>
httpClient.close();
{% endhighlight %}

当HttpClient实例不再使用时（比如应用shutdown、servlet被destroy时），注意记得close，它会shutdown内部的CM，以确保所有的connections资源及时得到释放。依靠java的垃圾回收来释放底层的连接资源不能保证时间点。

#### 多线程并发请求接口以及数据合并
{% highlight java linenos %}
public Map<Request, Result> multiGet(Request[] requests) {
    HttpGet[] httpGets = new HttpGet[requests.length];
    for (int i = 0; i < requests.length; i++) {
        httpGets[i] = buildHttpGet(requests[i]);
    }
    Map<Request, Result> result = new HashMap<Request, Result>();

    FutureTask<Result>[] futureTask = new FutureTask[requests.length];
    Thread[] threads = new Thread[requests.length];
    for (int i = 0; i < requests.length; i++) {
        futureTask[i] = new FutureTask<Result>(new MyCallable(this.client,httpGets[i],i + 1));
        threads[i] = new Thread(futureTask[i]);
    }

    // start the threads
    for (int j = 0; j < threads.length; j++) {
        threads[j].start();
    }

    // join the threads
    for (int j = 0; j < threads.length; j++) {
        try {
            result.put(requests[j], futureTask[j].get());
        } catch (ExecutionException e) {
            result.put(requests[j], new Result(0,""));
        } catch (InterruptedException e) {
            //what should do when interupted?
            result.put(requests[j], new Result(0,""))
        }
    }
    return result;
}
{% endhighlight %}

上面的代码是我封装的多线程并发执行多个http请求的multiGet方法，因为要在主线程的stack空间上得到各个子线程的请求响应数据，用了FutureTask来同步。

MyCallable实现了Callable接口，里面是执行请求的逻辑，不多做解释了，见下：
{% highlight c linenos %}
static class MyCallable implements Callable {
    private final CloseableHttpClient httpClient;
    private final HttpGet httpGet;
    private int id;

    public MyCallable(CloseableHttpClient httpClient, HttpGet httpGet, int id) {
        this.httpClient = httpClient;
        this.httpGet = httpGet;
        this.id = id;
    }
    @Override
    public Result call() {
        long begin = System.currentTimeMillis();
        try {
            ResponseHandler<String> responseHandler = new ResponseHandler<String>() {
                @Override
                public String handleResponse(final HttpResponse response) throws IOException {
                    int status = response.getStatusLine().getStatusCode();
                    if (status >= 200 && status < 300) {
                        HttpEntity entity = response.getEntity();
                        return entity != null ? EntityUtils.toString(entity) : null;
                    } else {
                        throw new ClientProtocolException("Unexpected response status: " + status);
                    }
                }

            };
            String ret = this.httpClient.execute(this.httpGet, responseHandler);
            int time = (int)(System.currentTimeMillis() - begin);
            return new Result(time, ret);
        } catch (ClientProtocolException e) {
            //todo
        } catch (IOException e) {
            //todo
        } finally {
            this.httpGet.reset();
        }
        return new Result((int)(System.currentTimeMillis() - begin), "");
    }
}
{% endhighlight %}

### **connection eviction**

前面提到，HttpClient是基于blocking I/O模型的，一个主要的缺点就是：任何socket fd上有I/O事件发生时，应用程序不一定能及时得知，只有阻塞在对该socket进行I/O操作时才知道。这就导致了一个很常见的问题：比如后端server将一个长连接关了后，客户端不能及时得知，只有等到客户端有线程去租到这个connection准备读写时发现抛异常了。

HttpClient通过在把底层连接租给线程前判断下连接的状态（底层socket的TCP状态）来缓解这个问题，然而这个判断状态的结果并不是100%可靠的，试想一个后端server不声不响的突然down了，TCP手都还没来得及挥，客户端如何能在短时间内知道连接的另一头是否还在呢。那么怎么办呢，HttpClient建议是开一个monitor线程去定期调cm.closeExpiredConnections()和cm.closeIdleConnections()去关掉一些过期连接和给定时间内不活跃的连接。

## 总结与讨论
对背景里描述的问题利用多线程并发访问的方式进行了优化，得到的性能提升也基本符合预期，即总的响应时间稍多于最长请求的响应时间。最后抛出个问题，针对背景里提出的问题有没有更好的解决方法？

