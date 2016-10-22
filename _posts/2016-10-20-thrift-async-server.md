---
layout: post
title: "thrift async server"
description: ""
category: 
tags: []
---
# 背景
在[Thrift Server](http://kapsterio.github.io/test/2016/07/06/ttheadedselectorserver.html)一文中，我结合源码介绍了thrift java lib里的几种server的，并详细介绍了TThreadedSelectorServer的实现。TThreadedSelectorServer将网络I/O和业务逻辑的执行分开，网络I/O基于异步事件驱动实现，而业务逻辑还是基于一组业务线程池，以阻塞、同步的方式执行。在thrift 0.9.1中引入了async processor，源自这个[issue](https://issues.apache.org/jira/browse/THRIFT-1972)。基于这个feature可以实现一个纯异步、事件驱动的server。相比于半同步、半异步的server，纯异步server将可以带来非常可观的性能提升。

# 同步Processor
先了解下thrift的TBaseProcessor，它的process方法是同步阻塞的，内部有个processMap: Map<String,ProcessFunction>，装着所有ProcessFunction，每个processFunction对应着一个业务接口。另外它还有一个实际业务handler的实例。

它的process方法代码如下：

<!--more-->

{% highlight java linenos %}
public boolean process(TProtocol in, TProtocol out) throws TException {
    TMessage msg = in.readMessageBegin();
    ProcessFunction fn = processMap.get(msg.name);
    if (fn == null) {
      TProtocolUtil.skip(in, TType.STRUCT);
      in.readMessageEnd();
      TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
      out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
      x.write(out);
      out.writeMessageEnd();
      out.getTransport().flush();
      return true;
    }
    fn.process(msg.seqid, in, out, iface);
    return true;
}
{% endhighlight %}

可以看出process实际上委托给map中的一个processFunction来执行，ProcessFunction的定义如下：

{% highlight java linenos %}
ProcessFunction<I, T extends TBase> 
{% endhighlight %}

这里的I会被具体化为实际的业务handler类，T则是handler类对应接口的入参类。

ProcessFunction的process方法代码如下：

{% highlight java linenos %}
public final void process(int seqid, TProtocol iprot, TProtocol oprot, I iface) throws TException {
    T args = getEmptyArgsInstance(); //一个抽象方法，由动态生成的子类实现
    try {
      args.read(iprot); //从 inprotocal中读取调用参数数据（反序列化）
    } catch (TProtocolException e) {
      iprot.readMessageEnd();
      TApplicationException x = new TApplicationException(TApplicationException.PROTOCOL_ERROR, e.getMessage());
      oprot.writeMessageBegin(new TMessage(getMethodName(), TMessageType.EXCEPTION, seqid));
      x.write(oprot);
      oprot.writeMessageEnd();
      oprot.getTransport().flush();
      return;
    }
    iprot.readMessageEnd(); 
    TBase result = getResult(iface, args); //抽象方法，阻塞地调用iface（实际的handler）的某个方法（不同的ProcessFunction子类调用代表自己的方法），得到对象方法的返回。
    //后面是将结果按照ooutprotocal协议序列化到outtransport中
    oprot.writeMessageBegin(new TMessage(getMethodName(), TMessageType.REPLY, seqid));
    result.write(oprot);
    oprot.writeMessageEnd();
    oprot.getTransport().flush(); 
}
{% endhighlight %}

可以看出，对实际业务handler的每个接口，thrift都将会生成一个ProcessFunction的实现类，并实现getEmptyArgsInstance方法和getResult方法。getEmptyArgsInstance作用是初始化一个handler接口入参对象。getResult会先初始化一个handler接口出参对象，然后去调用handler对应方法得到结果，把结果赋给出参对象的success属性。

下面将以campaignuser服务为例，看看实际生成的Processor和ProcessFunction是什么样子的。
campaignuser的部分idl如下：

{% highlight thrift linenos %}
service CampaignUserThriftService {
    map<i32,string> listStrategies(),

    map<i32,string> listClassifications(1: i32 strategyID),
}
{% endhighlight %}

下面是listClassifications接口对应的ProcessFunction实现类：
{% highlight java linenos %}
//可以看出在这里将ProcessFuncrion中的T给具体化了，对I增加了限制（为什么不把它具体化为Iface呢）
private static class listClassifications<I extends Iface> extends org.apache.thrift.ProcessFunction<I, listClassifications_args> {
      public listClassifications() {
        super("listClassifications");
      }

      protected listClassifications_args getEmptyArgsInstance() {
        return new listClassifications_args();
      }

      protected listClassifications_result getResult(I iface, listClassifications_args args) throws org.apache.thrift.TException {
        listClassifications_result result = new listClassifications_result();
        result.success = iface.listClassifications(args.strategyID);
        return result;
      }
}
{% endhighlight %}

# 异步Processor
thrift0.9.1中通过增加AsyncProcessor支持了纯异步的server，详见这个[issue](https://issues.apache.org/jira/browse/THRIFT-1972)。
在java的lib里，其中相比于同步TBaseProcessor增加了TBaseAsyncProcesser；增加了AsyncProcessFunction
以及一个AsyncFrameBuffer对应于原来的FrameBuffer。idl的compiler也相应增加了对AsyncProcessor的支持。

下面我们分别介绍为支持纯异步server涉及到的几个类的代码。

## TThreadedSelectorServer的变动
原来TThreadedSelectorServer的Selector线程在处理新连接请求时直接new出来一个FrameBuffer（见registerAccepted方法），现在则是调用createFrameBuffer，从而根据Processor是同步还是异步来决定构造FrameBuffer，代码见：

{% highlight java linenos %}
protected FrameBuffer createFrameBuffer(final TNonblockingTransport trans,
        final SelectionKey selectionKey,
        final AbstractSelectThread selectThread) {
        return processorFactory_.isAsyncProcessor() ?
                  new AsyncFrameBuffer(trans, selectionKey, selectThread) :
                  new FrameBuffer(trans, selectionKey, selectThread);
}
{% endhighlight %}

## TBaseAsyncProcessor
TBaseAsyncProcessor实现了TAsyncProcessor接口，TAsyncProcessor相对于TProcessor不同的地方在于process方法的定义：
{% highlight java linenos %}
public boolean process(final AsyncFrameBuffer fb) throws TException;

public boolean process(TProtocol in, TProtocol out) throws TException;
{% endhighlight %}

上面的是TAsyncProcessor的process方法，入参是个AsyncFrameBuffer，TBaseAsyncProcessor的实现如下：
{% highlight java linenos %}
public boolean process(final AsyncFrameBuffer fb) throws TException {

        //通过AsyncFrameBuffer得到输入和输出
        final TProtocol in = fb.getInputProtocol();
        final TProtocol out = fb.getOutputProtocol();

        //Find processing function
        final TMessage msg = in.readMessageBegin();
        AsyncProcessFunction fn = processMap.get(msg.name);
        if (fn == null) {
            TProtocolUtil.skip(in, TType.STRUCT);
            in.readMessageEnd();
            if (!fn.isOneway()) {
              TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
              out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
              x.write(out);
              out.writeMessageEnd();
              out.getTransport().flush();
            }
            fb.responseReady();
            return true;
        }

        //Get Args
        TBase args = fn.getEmptyArgsInstance();

        try {
            args.read(in); //和原来一样从in里读取数据（反序列化）到args对象
        } catch (TProtocolException e) {
            in.readMessageEnd();
            if (!fn.isOneway()) {
              TApplicationException x = new TApplicationException(TApplicationException.PROTOCOL_ERROR, e.getMessage());
              out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
              x.write(out);
              out.writeMessageEnd();
              out.getTransport().flush();
            }
            fb.responseReady();
            return true;
        }
        in.readMessageEnd();

        if (fn.isOneway()) { //如果是单向方法（没有返回值）那么读完入参后就可以通知framebuffer状态改变了。此时业务逻辑还完全没有执行
          fb.responseReady();
        }

        //start off processing function
        //这里的写法比较奇怪，既然resultHandler是fn的属性，为啥还要先将其取出来，再传给fn，直接将fb和seqid传给start方法不就ok了么~~
        AsyncMethodCallback resultHandler = fn.getResultHandler(fb, msg.seqid);
        try {
            //这里调用fn的start，那么start的作用应该是要调用iface的对应的业务逻辑方法（要求是非阻塞的），fn里start是抽象的，交由具体子类实现。
          fn.start(iface, args, resultHandler);
        } catch (Exception e) {
          resultHandler.onError(e);
        }
        return true;
}
{% endhighlight %}
注意到这个process是非阻塞的。

## AsyncFrameBuffer
AsyncFrameBuffer继承自FrameBuffer，不同的是它的invoke不会阻塞invokers线程，因为它调用的是TAsyncProcessor的process。代码如下：
{% highlight java linenos %}
public void invoke() {
      frameTrans_.reset(buffer_.array());
      response_.reset();

      try {
          //这里的eventHandler是干嘛用的？貌似是其他版本引入的
        if (eventHandler_ != null) {
          eventHandler_.processContext(context_, inTrans_, outTrans_);
        }
        ((TAsyncProcessor)processorFactory_.getProcessor(inTrans_)).process(this);
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
那剩下的问题就是原来的responseReady在哪执行？这就得看看compiler生成的一个个AsyncProcessFunction子类了。还是以listClassifications接口为例，它对应的AsyncProcessFunction子类是：

{% highlight java linenos %}
 public static class listClassifications<I extends AsyncIface> extends org.apache.thrift.AsyncProcessFunction<I, listClassifications_args, Map<Integer,String>> {
      public listClassifications() {
        super("listClassifications");
      }

      public listClassifications_args getEmptyArgsInstance() {
        return new listClassifications_args();
      }

      public AsyncMethodCallback<Map<Integer,String>> getResultHandler(final AsyncFrameBuffer fb, final int seqid) {
        final org.apache.thrift.AsyncProcessFunction fcall = this;
        return new AsyncMethodCallback<Map<Integer,String>>() { 
          public void onComplete(Map<Integer,String> o) {
            listClassifications_result result = new listClassifications_result();
            result.success = o;
            try {
              fcall.sendResponse(fb,result, org.apache.thrift.protocol.TMessageType.REPLY,seqid);
              return;
            } catch (Exception e) {
              LOGGER.error("Exception writing to internal frame buffer", e);
            }
            fb.close();
          }
          public void onError(Exception e) {
            byte msgType = org.apache.thrift.protocol.TMessageType.REPLY;
            org.apache.thrift.TBase msg;
            listClassifications_result result = new listClassifications_result();
            {
              msgType = org.apache.thrift.protocol.TMessageType.EXCEPTION;
              msg = (org.apache.thrift.TBase)new org.apache.thrift.TApplicationException(org.apache.thrift.TApplicationException.INTERNAL_ERROR, e.getMessage());
            }
            try {
              fcall.sendResponse(fb,msg,msgType,seqid);
              return;
            } catch (Exception ex) {
              LOGGER.error("Exception writing to internal frame buffer", ex);
            }
            fb.close();
          }
        };
      }

      protected boolean isOneway() {
        return false;
      }

      public void start(I iface, listClassifications_args args, org.apache.thrift.async.AsyncMethodCallback<Map<Integer,String>> resultHandler) throws TException {
        iface.listClassifications(args.strategyID,resultHandler);
      }
}
{% endhighlight %}
可以看出，它在getResultHandler里new出了一个AsyncMethodCallback<R>，这个callback的onComplete方法里会调用AsyncProcessFunction的sendResponse方法，这里面去responseReady:
{% highlight java linenos %}
public void sendResponse(final AbstractNonblockingServer.AsyncFrameBuffer fb, final TSerializable result, final byte type, final int seqid) throws TException {
        TProtocol oprot = fb.getOutputProtocol();

        oprot.writeMessageBegin(new TMessage(getMethodName(), type, seqid));
        result.write(oprot);
        oprot.writeMessageEnd();
        oprot.getTransport().flush();

        fb.responseReady();
}
{% endhighlight %}
那么谁来调用callback的onComplete呢？答案很简单，就是业务方法自己，在上面这个例子中iface.listClassifications(args.strategyID,resultHandler)实现时可以采用CompletableFuture或者RxJava等技巧实现异步回调通知resultHandler。

## 待续
我将在下一篇blog中结合一个proxy server的例子，介绍下async processor的使用，以及与原来基于同步processor的server在各个场景中做下性能对比。