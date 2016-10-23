---
layout: post
title: "thrift async client"
description: ""
category: 
tags: []
---
# 背景
最近对服务的逻辑进行了一些异步化改造，对依赖的thrift服务都改为异步调用，结合java8里提供的CompletableFuture，可以很方便地描述服务间的依赖关系。过程中对thrift的异步client实现细节产生兴趣，于是就有了这篇blog。
在[Thrift Server](http://kapsterio.github.io/test/2016/07/06/ttheadedselectorserver.html)一文中，我结合源码剖析了thrift的server组件中最常见的TThreadedSelectorServer的设计和实现。本文则关注的是thrift中提供了client。同步client非常简单，没什么值得研究的地方，因此本文将结合源码分析下异步client的实现。

<!--more-->

# thrift async client整体介绍和边界
thrift会将idl文件按照指定语言生成一个AsyncClient类(是TAsyncClient的子类)，每个接口都对应其中的一个方法。在这些方法里做的事情都类似，即先创建对应的TAsyncMethodCall子类的实例，然后交给一个TAsyncClientManager实例来完成实际RPC通信。

举个例子，比如我现在有个service的idl中定义了一个无参的listStrategies方法，那么在生成的AsyncClient类中就有个同名的方法，如下：
{% highlight java linenos %}
public void listStrategies(org.apache.thrift.async.AsyncMethodCallback<listStrategies_call> resultHandler) throws org.apache.thrift.TException {
      checkReady();
      listStrategies_call method_call = new listStrategies_call(resultHandler, this, ___protocolFactory, ___transport);
      this.___currentMethod = method_call;
      ___manager.call(method_call);
}
{% endhighlight %}
即生成的listStrategies多了个resultHandler参数，用于远程调用有结果时回调通知使用者

这是TAsyncClient的构造方法：
{% highlight java linenos %}
public TAsyncClient(TProtocolFactory protocolFactory, TAsyncClientManager manager, TNonblockingTransport transport, long timeout) {
    this.___protocolFactory = protocolFactory;
    this.___manager = manager;
    this.___transport = transport;
    this.___timeout = timeout;
}
{% endhighlight %}
一般来说，使用者每当发起一个RPC调用时，都需要创建一个TAsyncClient子类的实例，传入protocalFactory（确定所使用的protocal）、TAsyncClientManager实例（管理和远程service的实际I/O）、transport（底层socket连接，一般是从一个连接池中取到）以及指定下这次调用的timeout时间。

也就是说，从使用者角度来看，我们所需要为每个service构造一个clientManager、管理一个TNonblockingTransport的连接池，每次rpc调用先从连接池中取到一个连接，构造一个TAsyncClient子类的实例，然后调用其对应的方法并传入一个callback，执行rpc。

从TAsyncClient内部实现角度来看，本文主要关注以下几个问题：

- TAsyncClientManager内部是怎么管理多个pending call的
- TAsyncClientManager是怎么对后端接口设置超时的
- TAsyncMethodCall内部状态机迁移

# TAsyncClientManager内部是怎么管理多个pending call
TAsyncClientManager需要支持多个caller线程同时向远程service发起异步调用，因此它内部有个ConcurrentLinkedQueue<TAsyncMethodCall>作为异步调用的队列，caller线程只需要准备好异步调用的相关数据，然后把TAsyncMethodCall实例入队，并唤醒selector就可以了。代码在TAsyncClientManager的call方法里：
{% highlight java linenos %}
public void call(TAsyncMethodCall method) throws TException {
    if (!isRunning()) {
      throw new TException("SelectThread is not running");
    }
    method.prepareMethodCall();
    pendingCalls.add(method);
    selectThread.getSelector().wakeup();
}
{% endhighlight %}

selector在每轮eventloop中从pending call队列中循环地poll，取出所有的TAsyncMethodCall实例，加到一个TreeSet中（下一节介绍用途），并将这个调用使用的transport注册到selector中去，并初始化这个调用的内部状态机状态。代码如下：
{% highlight java linenos %}
private void startPendingMethods() {
      TAsyncMethodCall methodCall;
      while ((methodCall = pendingCalls.poll()) != null) {
        // Catch registration errors. method will catch transition errors and cleanup.
        try {
          methodCall.start(selector);

          // If timeout specified and first transition went smoothly, add to timeout watch set
          TAsyncClient client = methodCall.getClient();
          if (client.hasTimeout() && !client.hasError()) {
            timeoutWatchSet.add(methodCall);
          }
        } catch (Exception exception) {
          LOGGER.warn("Caught exception in TAsyncClientManager!", exception);
          methodCall.onError(exception);
        }
      }
}
{% endhighlight %}

# TAsyncClientManager是怎么对后端接口设置超时的
在TAsyncClientManager内部的SelectThread中有个TreeSet存放着已经发起remote请求的调用，按照每个call的timeout时间戳升序排序。我们知道TreeSet内部是个红黑树，基本上所有操作都是O(lgn)的。这样一来，selector线程在每轮的eventloop中select时就可以传入一个timeout参数，值为最近要超时的call的超时时间戳减去当前时间。在eventloop中超时处理方法里就可以对每个已超时的call做超时处理。selector线程的事件循环代码如下：
{% highlight java linenos %}
 public void run() {
      while (running) {
        try {
          try {
            if (timeoutWatchSet.size() == 0) {
              // No timeouts, so select indefinitely
              selector.select();
            } else {
              // We have a timeout pending, so calculate the time until then and select appropriately
              long nextTimeout = timeoutWatchSet.first().getTimeoutTimestamp();
              long selectTime = nextTimeout - System.currentTimeMillis();
              if (selectTime > 0) {
                // Next timeout is in the future, select and wake up then
                selector.select(selectTime);
              } else {
                // Next timeout is now or in past, select immediately so we can time out
                selector.selectNow();
              }
            }
          } catch (IOException e) {
            LOGGER.error("Caught IOException in TAsyncClientManager!", e);
          }
          transitionMethods();
          timeoutMethods();
          startPendingMethods();
        } catch (Exception exception) {
          LOGGER.error("Ignoring uncaught exception in SelectThread", exception);
        }
      }
}
{% endhighlight %}
超时处理方法代码如下：
{% highlight java linenos %}
private void timeoutMethods() {
      Iterator<TAsyncMethodCall> iterator = timeoutWatchSet.iterator();
      long currentTime = System.currentTimeMillis();
      while (iterator.hasNext()) {
        TAsyncMethodCall methodCall = iterator.next();
        if (currentTime >= methodCall.getTimeoutTimestamp()) {
          iterator.remove();
          methodCall.onError(new TimeoutException("Operation " + methodCall.getClass() + " timed out after " + (currentTime - methodCall.getStartTime()) + " ms."));
        } else {
          break;
        }
      }
}
{% endhighlight %}

# TAsyncMethodCall内部状态机迁移

- prepareMethodCall:数据准备，把要传给远程方法的参数对象按照使用的协议写入一段TMemoryBuffer中，然后计算数据size，初始化frameBuffer和sizeBuffer

- start：状态机启动，由selector线程在每轮的eventloop中startPendingMethods方法中调用，将当前的transport注册到selector中。

- transition：事件发生时的状态迁移，由selector线程在每轮的eventloop中transitionMethods方法里调用。代码很直观，如下：

{% highlight java linenos %}
protected void transition(SelectionKey key) {
    // Ensure key is valid
    if (!key.isValid()) {
      key.cancel();
      Exception e = new TTransportException("Selection key not valid!");
      onError(e);
      return;
    }

    // Transition function
    try {
      switch (state) {
        case CONNECTING:
          doConnecting(key);
          break;
        case WRITING_REQUEST_SIZE:
          doWritingRequestSize();
          break;
        case WRITING_REQUEST_BODY:
          doWritingRequestBody(key);
          break;
        case READING_RESPONSE_SIZE:
          doReadingResponseSize();
          break;
        case READING_RESPONSE_BODY:
          doReadingResponseBody(key);
          break;
        default: // RESPONSE_READ, ERROR, or bug
          throw new IllegalStateException("Method call in state " + state
              + " but selector called transition method. Seems like a bug...");
      }
    } catch (Exception e) {
      key.cancel();
      key.attach(null);
      onError(e);
    }
}
{% endhighlight %}

在doReadingResponseBody方法里，当全部数据都读完后调用callback的onComplete方法。中间有任何异常发生时，关闭transport，并调用callback的onError方法。从这里也可以看出，callback的onComplete方法、onError方法里不能有阻塞当前线程的操作。

# 总结
异步客户端实现很简单，最重要的一点是了解所有调用的callback都会在同一个selector线程里执行，因此onComplete和onError不能有阻塞当前线程的操作。下一篇blog中，我将结合源码了解下thrift0.9.1中引入的纯异步server。源自这个[issue](https://issues.apache.org/jira/browse/THRIFT-1972)