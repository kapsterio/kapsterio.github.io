---
layout: post
title: "Thrift Server —— TTheadedSelectorServer"
description: ""
category: 
tags: []
---
# 背景
Thrift是一个由FB在09年开源的序列化/RPC框架，具有优秀的跨语言特性，现在已经成为apache基金会的子项目。这里先对Thrift本身做个简单介绍，其主要包括三个组件：protocol，transport和server，其中，protocol定义了消息是怎样序列化的，transport定义了消息是怎样在客户端和服务器端之间通信的，server用于从transport接收序列化的消息，根据protocol反序列化之，调用用户定义的消息处理器，并序列化消息处理器的响应，然后再将它们写回transport。Thrift模块化的结构使得它能提供各种server实现，有如下几个：

- TSimpleServer
- TThreadPoolServer
- TNonblockingServer
- THsHaServer(half-Sync/Half-Async：半同步、半异步)
- TThreadedSelectorServer

<!--more-->

前两种是阻塞I/O型的，后三者是基于Non-Blocking I/O。这篇[文章](https://github.com/m1ch1/mapkeeper/wiki/Thrift-Java-Servers-Compared)通俗易懂地介绍了这五种Thrift Server的功能和特性，并做了一些性能比较。

本文主要关注的是生产环境中最常用的TThreadedSelectorServer（公司的MTThrift framework采用的就是这个）并结合源码详细介绍他的设计和实现，顺便学习下java NIO的一些最佳实践

# TTHreadedSelectorServer
首先简单介绍下本文的主角，它自Thrift 0.8版本后引入。类似THsHaServer，它也是一种半同步半异步Server。与THsHaServer不同的是，TTHreadedSelectorServer处理网络I/O的线程不再是单独的一个，而是一组。在处理网络I/O这组线程中，一个线程专门epoll listeningSocket（accept）方便起见称之为acceptor。剩下的线程数量可配置，负责epoll connectionSocket (read and write)，称之为selectors（为了方便起见，这里使用epoll代指在多个网络连接上多路复用）。另外，它也有另一组线程来执行实际的业务逻辑，称之为业务线程组（invokers）。

acceptor接受client端的连接请求，并以round-robin方式选择一个selector线程，将accept到的connection注册给这个selector线程，并由这个selector线程负责处理在这个connection上发生的read、write网络I/O事件。

selector线程的工作可以分为两部分。首先当acceptor交给它一个connection时，它监听着这个connection上的read事件，并为之attach一个状态机标识目前所处的thrift数据帧读取状态，当读完一次完整的thrift请求数据时，它构造一个任务并交给业务线程组去执行，这个任务干的事就是首先执行TProcessor的process方法，也就是实际的业务逻辑，然后通知selector线程：我的活干完了，剩下的活你继续。好的，接下来就是selector的第二部分工作，将任务执行结果数据write给connection。

以上就是TTHreadedSelectorServer内部几种线程的分工，职责明确、简单清晰。下面将结合源码回答如下几个问题：
- acceptor如何选择selector线程来做负载均衡
- acceptor接受连接请求的策略是什么
- acceptor怎么把一个connection注册给某个selector
- selector是怎么处理网络I/O（包括：读写网络数据、状态机维护）
- selector是怎么构建一个任务的
- invoker在执行完任务后怎么告知selector

 以上问题基本上涵盖了TTHreadedSelectorServer的所有设计细节。下面先给出一张整体的示意图：
 ![thrift-server](/public/fig/server.png)

## acceptor怎么选择selector线程
当client端一个连接请求过来后，acceptor先accept得到一个connection，然后需要选择一个selector线程来处理这个connection。目前的做法很简单，acceptor有个属性——一个SelectorThreadLoadBalancer对象，代码如下：
{% highlight java linenos %}
protected class SelectorThreadLoadBalancer {
    private final Collection<? extends SelectorThread> threads;
    private Iterator<? extends SelectorThread> nextThreadIterator;

    public <T extends SelectorThread> SelectorThreadLoadBalancer(Collection<T> threads) {
      if (threads.isEmpty()) {
        throw new IllegalArgumentException("At least one selector thread is required");
      }
      this.threads = Collections.unmodifiableList(new ArrayList<T>(threads));
      nextThreadIterator = this.threads.iterator();
    }

    public SelectorThread nextThread() {
      // Choose a selector thread (round robin)
      if (!nextThreadIterator.hasNext()) {
        nextThreadIterator = threads.iterator();
      }
      return nextThreadIterator.next();
    }
  }
{% endhighlight %}
通过调用nextThread（）方法就可以以round-robin方式遍历所有的selector线程，如果后续想要实现其他负载均衡策略，只需要改下SelectorThreadLoadBalancer的实现，比如可以让每个selector维护一个负责的connection数，在nextThread()方法选择连接数最小的那个selector返回。

## acceptor接受请求的策略
acceptor在选择完一个selector后，有两种策略将connection交给这个selector线程，分别是AcceptPolicy.FAST_ACCEPT和FAIR_ACCEPT。前一种FAST_ACCEPT很简单，意为尽快交付给selector，会在acceptor线程中执行交付动作。后一种则会牵扯到业务线程组(invokers)，构造了一个交付任务，抛给invokers，让它来执行交付动作，目的是考虑到业务线程组的处理能力。比如，当服务的业务逻辑比较重，业务线程组处理不过来，如果此时acceptor还将connection交付给selector，selector底层是epoll，处理起来很快，很快就把请求数据从操作系统的TCP read buffer读取、并解析到应用程序内存中（JVM堆中），然后再构造任务交给invokers，又加重了invokers的负担。造成的结果就是任务的积压，大量占用应用程序内存。如果acceptor把交付动作交给invokers来执行，这个问题就得以解决。这部分的代码如下：
{% highlight java linenos %}
if (args.acceptPolicy == Args.AcceptPolicy.FAST_ACCEPT || invoker == null) {
          doAddAccept(targetThread, client);
} else {
    // FAIR_ACCEPT
    try {
        invoker.submit(new Runnable() {
              public void run() {
                doAddAccept(targetThread, client);
              }
        });
    } catch (RejectedExecutionException rx) {
        LOGGER.warn("ExecutorService rejected accept registration!", rx);
        // close immediately
        client.close();
    }
}
{% endhighlight %}

## acceptor怎么把一个connection注册给某个selector
这个问题涉及到connection在不同线程之间的移交，TTheadedSelectorServer的selector都有个acceptedQueue（一个装着connection的blockingQueue）。acceptor向selector注册connection只是将connection简单地添加到selector的acceptedQueue中，代码如下：
{% highlight java linenos %}
public boolean addAcceptedConnection(TNonblockingTransport accepted) {
      try {
        acceptedQueue.put(accepted);
      } catch (InterruptedException e) {
        LOGGER.warn("Interrupted while adding accepted connection!", e);
        return false;
      }
      selector.wakeup(); //唤醒阻塞在select()上的线程
      return true;
}
{% endhighlight %}

然后由selector在每轮的事件循环将acceptedQueue内容poll出来，注册到NIO的事件循环中，代码如下：
{% highlight java linenos %}
private void processAcceptedConnections() {
      // Register accepted connections
      while (!stopped_) {
        TNonblockingTransport accepted = acceptedQueue.poll();
        if (accepted == null) {
          break;
        }
        registerAccepted(accepted);
      }
}

private void registerAccepted(TNonblockingTransport accepted) {
      SelectionKey clientKey = null;
      try {
        clientKey = accepted.registerSelector(selector, SelectionKey.OP_READ);
        FrameBuffer frameBuffer = new FrameBuffer(accepted, clientKey, SelectorThread.this);
        clientKey.attach(frameBuffer);
      } catch (IOException e) {
        LOGGER.warn("Failed to register accepted connection to selector!", e);
        if (clientKey != null) {
          cleanupSelectionKey(clientKey);
        }
        accepted.close();
      }
    }
}
{% endhighlight %}
在registerAccepted方法中执行注册逻辑，可以看出，它一上来就监听connection上的读事件，并且为得到的SelectionKey对象attach了一个frameBuffer（关于java NIO的SelectionKey类的说明可以读读它的[javadoc](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/SelectionKey.html)，个人觉得SelectionKey和epoll里面的epoll_event底层应该是一回事）。frameBuffer很关键，它是读写网络数据的关键数据结构，保存着状态机、从网络下读下来的thrift协议数据、业务线程组执行任务的结果（响应给client的数据），详细内容见下节。

## selector是怎么处理网络I/O
当selector监听的某个connection可读时，handleRead方法被调用，代码如下：
{% highlight java linenos %}
protected void handleRead(SelectionKey key) {
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (!buffer.read()) {
        cleanupSelectionKey(key);
        return;
      }

      // if the buffer's frame read is complete, invoke the method.
      if (buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
          cleanupSelectionKey(key);
        }
      }
}
{% endhighlight %}
可以看出，它调用frameBuffer的read方法来执行实际的read动作读取数据，当判断已经读完数据时，调用requestInvoke方法来构造一个任务表示服务的实际业务逻辑，让业务线程组去执行，本节主要关注怎么去网络上读写数据以及状态机维护，构造任务并执行的逻辑见下一节。

### 状态机
frameBuffer内部维护一个状态机，根据目前所处的状决定执行不同的动作。共有如下几个状态：

- READING_FRAME_SIZE, 标识目前处于正在读取一个thrift完整帧的frame_size
- READING_FRAME, 标识正在读取thrift帧内容数据
- READ_FRAME_COMPLETE,  标识已经完整地读取了一帧thrift数据
- AWAITING_REGISTER_WRITE, 标识正在等待切换connection上感兴趣的I/O事件类型（切换成监听可写事件）
- WRITING, 标识正在向网络写数据
- AWAITING_REGISTER_READ, 标识正在等待切换connection上感兴趣的I/O事件类型（切换成监听可读事件）
- AWAITING_CLOSE，标识正在等待把connection给close掉

状态迁移转换如下所示:
 ![thrift-server](/public/fig/state.png)

从状态迁移逻辑我们可以看出，thrift协议数据帧可以分为两个部分：

 -----------------------
     size (4 bytes)

 -----------------------
    frame data(size bytes)

### ByteBuffer
从connection上读、以及向connection写数据都需要用到ByteBuffer类，ByteBuffer的具体使用见[javadoc](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)。值得一提的是ByteBuffer类存在两个不同的分配内存的方法： allocate和allocateDirect，分别得到就是non-direct buffer和direct buffer。二者区别在于non-direct buffer位于JVM堆中，数据的arrangement、对齐等等格式由具体JVM决定，direct buffer虽然也在应用程序内存空间中，但不位于JVM的堆中，因此也就不会自动被GC，是程序直接向操作系统申请分配得到的内存空间，自然地，数据的arrangement、对齐等等也就由操作系统说了算了。那么direct buffer有什么好处呢？java中，在进行一些I/O操作时往往需要涉及到位于JVM堆的内存数据和网络、磁盘数据之间通过系统调用进行读写，这个过程都是需要java去分配一个临时的direct buffer来作为辅助的，数据都需要经过这个临时的direct buffer倒腾一趟，因为系统调用只认操作系统的数据arrangement、对齐方式。如果使用allocateDirect直接分配一个direct buffer，后续在进行I/O操作时也就是不需要临时的direct buffer了，效率会有很大提升。关于direct buffer想了解更多可以参考stackoverflow上[这个问题下面的问答](http://stackoverflow.com/questions/5670862/bytebuffer-allocate-vs-bytebuffer-allocatedirect)。

回到TTheadedSelectorServer上，它没有使用direct buffer(>_<)。TTheadedSelectorServer用maxReadBufferBytes参数来限制服务当前读取的thrift数据量大小，维护一个全局变量（AtomicLong readBufferBytesAllocated）标识已分配空间大小，当readBufferBytesAllocated大于maxReadBufferBytes时，则暂不分配空间来存放thrift的请求数据，那么这些数据将暂存于操作系统的TCP read buffer中。

当完全读完一帧thrift数据时，selector将无需再监听connection的可读事件，因此通过selectionKey完成这一操作，
{% highlight java %}
selectionKey_.interestOps(0);
{% endhighlight %}
java NIO中想要改变某个connection上感兴趣的I/O事件就是通过与之对应的SelectionKey的interestOps方法进行的。

详细的读写数据过程参见FrameBuffer的read和write方法。

## selector是怎么构建一个任务的
当selector已经完整地读取下来一帧thrift数据，存放在内存的buffer中，接下来就交给业务线程组来执行解析thrift协议，执行具体业务逻辑的事，执行完了后，业务线程组还需要通知selector线程执行完毕，可以去响应执行结果给client端了。这个过程被封装成一个任务，具体的逻辑见frameBuffer的invoke方法：
{% highlight java linenos %}
public void invoke() {
      TTransport inTrans = new TMemoryInputTransport(buffer_.array());
      TProtocol inProt = inputProtocolFactory_.getProtocol(inTrans);
      response_ = new TByteArrayOutputStream();
      TProtocol outProt = outputProtocolFactory_.getProtocol(new TIOStreamTransport(response_));

      try {
        processorFactory_.getProcessor(inTrans).process(inProt, outProt);
        responseReady();
        return;
      } catch (TException te) {
        LOGGER.warn("Exception while invoking!", te);
      } catch (Throwable t) {
        LOGGER.error("Unexpected throwable while invoking!", t);
      }
      // This will only be reached when there is a throwable.
      state_ = FrameBufferState.AWAITING_CLOSE;
      requestSelectInterestChange();
}
{% endhighlight %}

可以看出，它先从buffer_构造一个inputTransport，再从responese_这个输出字节流构造一个outputTransport（我觉得这里为了统一也可以从buffer_先构造一个inputStream，再从这个inputStream同样构造一个TIOStreamTransport作为inputTransport），然后在TProcessor的process方法中执行解析thrift协议、实际的业务逻辑（不需要我们操心了）。最后在responseReady方法里先把responese_赋给buffer_，以便后续select去write，然后通过requestSelectInterestChange方法通知selector线程去改变connection的感兴趣的I/O事件。

怎么去通知呢？其实Selector有个成员集合selectInterestChanges，它装着哪些需要改变I/O事件监听兴趣的frameBuffer，requestSelectInterestChange里只是简单地把frameBuffer添加到这个集合当中，由Selector在每轮的事件循环中遍历这个集合并处理，处理逻辑如下：
{% highlight java linenos %}
public void changeSelectInterests() {
      if (state_ == FrameBufferState.AWAITING_REGISTER_WRITE) {
        // set the OP_WRITE interest
        selectionKey_.interestOps(SelectionKey.OP_WRITE);
        state_ = FrameBufferState.WRITING;
      } else if (state_ == FrameBufferState.AWAITING_REGISTER_READ) {
        prepareRead();
      } else if (state_ == FrameBufferState.AWAITING_CLOSE) {
        close();
        selectionKey_.cancel();
      } else {
        LOGGER.error("changeSelectInterest was called, but state is invalid (" + state_ + ")");
      }
}
{% endhighlight %}

# 总结
java的NIO使用起来和epoll很类似，相比之下使用netty可能要更直观些，另外netty使用direct buffer来存放数据，会比目前thrift server这种实现更高效些。google的[grpc-java](https://github.com/grpc/grpc-java)的最重要的一种Transport就是基于netty来实现的。
关于grpc，扮演的角色和thrift这一套非常相似，文档也非常翔实，更多请参见它的[官网](http://www.grpc.io/)。







 

