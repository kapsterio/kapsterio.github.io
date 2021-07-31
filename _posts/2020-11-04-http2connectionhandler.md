---
layout: post
title: "netty http2 实现框架解读"
description: ""
category: 
tags: []
---

## Overview

### Http2ConnectionHandler

在上一篇blog已经大体介绍了。
> 设计上，他可以说主要是Inbound handler，继承自ByteToMessageDecoder，通过内部decoder来对http2协议进行decode，decoder内部再委托给使用者自己实现的Http2FrameListener对收到的http2消息进行业务处理。这里和netty中常见的decoder decode出协议封装成对象消息交给pipeline下游业务handler的做法不太一样。另外，他在实现上又实现了ChannelOutboundHandler接口，看上去主要是为了实现flush方法，并不是作为一个encoder handler。如果使用方想去向连接上响应http2的response还需要通过他的encoder()方法拿到一个Http2FrameWriter对象，通过Http2FrameWriter的各种writeXXX接口直接write对应的http2消息/帧，可以说非常原始了。

<!--more-->

### Http2ConnectionDecoder
Http2ConnectionHandler内部负责http2协议decode的接口。

注意他的作用不仅在于decode出协议frame，而且还会进行一些http2定义控制语义行为：比如流量控制、流优先级控制等等。做完这些http2协议定义的控制行为后将帧通过`Http2FrameListener`通知给上层业务（这里也不一定是传输层之上的上层业务，http2层面也有很多Http2FrameListener的实现）

另外一点就是：他不直接decoder http2协议数据，而是委托给内部的`Http2FrameReader`去做真正的解析工作

值得一提的Http2ConnectionHandler虽然是一个ByteToMessageDecoder，但是在其decode方法的实现中并没有将decode出的frame msg通过`out`形参传递给pipeline的下一个handler，而是调用`Http2FrameListener`对应的方法通知应用程序。


### Http2FrameReader
实际进行http2协议decode的接口，默认实现`DefaultHttp2FrameReader`在readFrame方法中对netty read得到的ByteBuf进行decode，实现上没有使用netty提供的LengthFieldBasedFrameDecoder, 自己通过do while循环实现帧解析，然后调用`Http2FrameListener`对应的方法。

内部实现就是一个简单的状态机，

### Http2FrameListener
前面已经多次提到，业务层需要实现的接口，用于接受http2的各种数据+控制帧


### Http2ConnectionEncoder
和`Http2ConnectionDecoder`基本对等，其作用也不仅在于encode frame，同时他也需要实现http2定义的控制语义行为。

为了做到这样，他在定义上继承了`Http2FrameWriter`接口，这样应用在有帧要发送时需要先经过他，执行完协议的控制行为后再通过调用内部的`Http2FrameWriter` 进行基本的encode & write。


### Http2FrameWriter
作用和`Http2FrameReader` + `Http2FrameListener` 组合基本对等，提供了各种writeXXX方法用于向connection上写入http2的各种frame。

默认实现`DefaultHttp2FrameWriter`中实现了所有writeXXX接口，做了最基本的http2协议encode、以及将encode得到的ByteBuf write给Channel。



### Http2Connection
对http2连接的抽象，内部主要管理连接上所有的流（Http2Stream）。
- 提供api根据stream id查询stream、遍历所有的active stream。
- 另外定义了一个stream生命周期实际的listener，提供增加删除listener的api。
- 定义Endpoint接口（某一端对connection的视图），提供返回local/remote视图的api


#### Endpoint
Connection在某一端的视图，是因为一个Connection两端都可以发起stream、两端也都有独立的状态数据
实现上它是Http2Connection的内部接口，能够访问Http2Connection的实现所有属性，同时提供一些创建stream相关的api，以及获取当前端flowController的api


#### Listener
Http2Connection内部定义的Listener接口是为了能够接受connection上stream的各种生命周期事件的通知。

在DefaultHttp2Connection的实现里，通过listeners list做到依次通知所有listener



### Http2FrameCodec
在上一篇blog也已经大体介绍了。

> 回到Http2FrameCodec本身，他作为http2 frame的codec，继承自Http2ConnectionHandler，负责对当前存活着的stream上frame进行decode和encode, 和下游handler之间通过Http2StreamFrame进行交互。

在实现上，他主要做了两件事
- inbound: 通过在内部的`Http2FrameListener` 实现的各个onXXXFrame方法里对各种frame数据进行封装，封装成对应的`Http2Frame`对象。然后向channel上将`Http2Frame`对象通过fireChannelRead fire给下一个handler。
- outbound: 在write方法里通过msg的类型(各种`Http2Frame`实现类)来决定调用内部`Http2ConnectionEncoder`相对应的writeXXX方法



### Http2MultiplexHandler
在上一篇blog也已经大体介绍了。

> 这个类主要是对http2 stream复用这个行为进行了封装，当连接上有新的stream创建时，为其创建一个新的Http2MultiplexHandlerStreamChannel，并注册到eventLoop中，并对应用代码的handler屏蔽或者转换一些原有连接上frame消息（主要是控制frame），使得应用handler面向新的Http2MultiplexHandlerStreamChannel编程，只需要关注子channel上的事件。使用上需要和Http2FrameCodec一起使用。

![interface](../draw.io/http2ConnectionHandler.svg)


## connection setup
todo



## Inbound数据处理

### Overview

```plantumlcode
@startuml

-> Http2ConnectionHandler: decode(...)

Http2ConnectionHandler -> DefaultHttp2ConnectionDecoder : decodeFrame(...)

DefaultHttp2ConnectionDecoder -> DefaultHttp2FrameReader : readFrame(...)

DefaultHttp2ConnectionDecoder --> FrameReadListener : onXXXRead(...)

FrameReadListener -> FrameReadListener : ...

participant "Http2FrameListener" as listener #99FF99

FrameReadListener -> listener: onXXXRead(...) (callback the application)

listener -> FrameReadListener: return

FrameReadListener -> FrameReadListener : ...

@enduml
```


`FrameReadListener` 也是一个`Http2FrameListener`， 他负责处理协议控制相关的逻辑（流量控制、stream状态切换、流优先级控制等等），另外负责调用应用层的listener执行应用程序的回调。


下面分http2协议帧类型，解读下`FrameReadListener`的实现。


### onDataRead
```plantumlcode
@startuml

-> FrameReadListener: onDataRead(...)

participant "DefaultHttp2LocalFlowController" as DefaultHttp2LocalFlowController #FF6600

FrameReadListener -> DefaultHttp2LocalFlowController : receiveFlowControlledFrame(...)


participant "FlowState (connectionState)" as connectionState

participant "FlowState (streamState)" as streamState

DefaultHttp2LocalFlowController -> connectionState : receiveFlowControlledFrame (connection-level)

connectionState -> connectionState: decrease the window

DefaultHttp2LocalFlowController -> streamState : receiveFlowControlledFrame (stream-level flow controll)

streamState -> streamState: decrease the window

participant "Http2FrameListener" as listener #99FF99


FrameReadListener -> listener: onDataRead(...) (callback the application)

listener -> FrameReadListener: bytesToReturn (retrieve the number of bytes that have been processed)


FrameReadListener -> DefaultHttp2LocalFlowController: consumeBytes(bytesToReturn, ...)


DefaultHttp2LocalFlowController -> connectionState: consumeBytes(...)

connectionState -> connectionState: decrease processedWindow

alt processedWindow <= 1/2 * initialWindow case

    connectionState -> connectionState: expand window back to intial window

    connectionState -> Http2FrameWriter : writeWindowUpdate(...) 
end

DefaultHttp2LocalFlowController -> streamState: consumeBytes(...)

streamState -> streamState: decrease processedWindow

alt processedWindow <= 1/2 * initialWindow case

    streamState -> streamState: expand window back to intial window

    streamState -> Http2FrameWriter : writeWindowUpdate(...) 
end

alt end of Stream case

    FrameReadListener -> Http2LifecycleManager : closeStreamRemote(...)

end 
@enduml

```

DATA帧是需要收flow controll的，因此onDataRead里涉及到`DefaultHttp2LocalFlowController`来进行流控处理。上图是一个收到一个DATA帧后的正常情况处理流程。

### DefaultHttp2LocalFlowController


作为http2接口端流控的基本实现，算法思想很简单： 就是在window 小于 windowUpdateRatio * initialWindow时发送WINDOW_UPDATE帧来告诉发送方扩大窗口继续发送。 

实现上主要维护两个值： window和processedWindow
- window: 实际的接受窗口大小，一旦收到data，就会去从window上减去对应大小
- processedWindow: window的视图，用于决定是否发送window_update，收到data后不会立即减少，而是在应用层的onDataRead返回已处理数据大小后，再去减小（减掉应用层onDataRead返回的值）

DATA帧也可能携带end of stream标示，此时会导致stream状态发生切换，FrameReadListener通过调用Http2LifecycleManager的closeStreamRemote做到状态切换，Http2LifecycleManager就是Http2ConnectionHandler本身。至于什么要在Http2ConnectionHandler实现Http2LifecycleManager, 后面再分析分析。

### onHeadersRead


```plantumlcode
@startuml

-> FrameReadListener: onHeadersRead(...)
FrameReadListener -> Http2Connection: stream(streamId)
Http2Connection -> FrameReadListener : return stream (Http2Stream)

alt stream not exist
    FrameReadListener -> Http2Connection: remote()
    Http2Connection -> FrameReadListener: return remote Endpoint (DefaultEndpoint)
    FrameReadListener -> DefaultEndpoint: createStream(streamId)
    DefaultEndpoint -> DefaultStream : new DefaultStream(streamId, state)
    DefaultEndpoint -> DefaultEndpoint :  incrementExpectedStreamId
    DefaultEndpoint -> DefaultEndpoint : addStream(stream)
    DefaultEndpoint -> DefaultStream : stream.activate()
else stream.state() is RESERVED_REMOTE
    FrameReadListener -> DefaultStream : stream.open()
end

participant "DefaultHttp2RemoteFlowController" as DefaultHttp2RemoteFlowController #FF6600

FrameReadListener -> DefaultHttp2RemoteFlowController : updateDependencyTree(...)


participant "StreamByteDistributor" as StreamByteDistributor #FF6600

DefaultHttp2RemoteFlowController -> StreamByteDistributor : updateDependencyTree(...)


participant "Http2FrameListener" as listener #99FF99

FrameReadListener -> listener: onHeadersRead(...)

@enduml
```

- 首先判断streamID对于的stream是否存在，不存在的话需要通过Remote Endpoint创建一个stream。创建stream会将其add到Http2Connection内部的streamMap中，同时执行那些注册在Http2Connection 里面的listener，
Http2Connection通过listener将stream的创建、关闭等等事件通知到FlowController、StreamByteDistributor等等组件，做到解耦
- http2Connection实现中内部除了有个保存所有stream的streamMap外还有一个保存活跃stream的ActiveStreams, 主要是为了支持forEachActiveStream操作
- 同样，如果headers帧包含end of stream标示，则会通过FrameReadListener通过调用Http2LifecycleManager的closeStreamRemote做到关闭流的remote端

后面将重点分析下`StreamByteDistributor`的实现


### onPushPromiseRead
和onHeadersRead类似，通过connection.remote().reservePushStream去预留一个stream，但是不会activate这个stream


### onPriorityRead
逻辑很简单，只需要通知encoder的flowController，去updateDependencyTree，然后通知应用层listener

### onRstStreamRead
逻辑也比较简单，先做些状态的校验，然后通知应用层listener，最后通过调用lifecycleManager.closeStream关闭stream

### onSettingsRead
协议规定收到SETTINGS帧后，一旦所有的配置值都处理完后，接收方必须立即回复SETTINGS ack，发送方收到ack后，就可以应用新的配置值了。在netty的实现中，除了遵守这个行为外，还想做到：settings的接收方需要在发送任何利用了新的settings参数的帧之前发送setting ack帧。因此先通过encoder.writeSettingsAck() 回复给发送方ack，然后通过encoder.remoteSettings()应用settings， remoteSettings里将settings设置应用到Http2FrameWriter.Configuration里。

比如SETTINGS帧里设置了INITIAL_WINDOW_SIZE, encoder.remoteSettings()会调用flowController().initialWindowSize()，这会导致flow controller触发writePendingBytes。因此需要前置writeSettingsAck()， 详见[issue 6520](https://github.com/netty/netty/issues/6520)，[issue 6521](https://github.com/netty/netty/pull/6521)。

### onSettingsAckRead
SETTINGS发送方在收到接收方的SETTINGS ACK后需要应用settings参数，由于可能有多个SETTINGS的存在，因此在netty DefaultHttp2ConnectionEncoder的实现中有个outstandingLocalSettingsQueue，有序存放还没有ack的settings，当收到一个ack时，从队尾取出并通过调用applyLocalSettings应用设置。

applyLocalSettings里将settings设置应用到Http2FrameReader.Configuration里（和remoteSettings对应）

http2协议里定义了如下几种SETTINGS:

- SETTINGS_HEADER_TABLE_SIZE: 发送方告诉远端自己header compression table的最大大小，默认值是4096字节
- SETTINGS_ENABLE_PUSH: 如果值为0，表示发送方不接受server push。应该只有client端才能去设置这个参数，一旦client端设置了这个参数，这个connection就是个普通的request-response式的单向连接通道。默认值为1
- SETTINGS_MAX_CONCURRENT_STREAMS: 用于发送方告诉远程自己允许对端并发创建的stream个数。client端如果将这个值设置为0，等同于关闭server push。server端应该尽可能避免将这个值设置为0（或者只维持很短的时间，比如负载高了）
- SETTINGS_INITIAL_WINDOW_SIZE：用于告诉对端自己的stream level的初始接收窗口大小，注意这个初始窗口大小是针对stream-level的，connection level的初始窗口大小是固定的65535，没得改，当然，可以通过window update去改connection level和stream level的窗口大小。默认值也是65535。
- SETTINGS_MAX_FRAME_SIZE： 用于告诉对端自己能够接收的最大帧大小，初始值是2^14 -1 (16384)
- SETTINGS_MAX_HEADER_LIST_SIZE： 用于告诉对端自己能够接受的最大的header list字节数。

### onPingRead
处理很简单，通过encoder.writePing()去回复一个ping ack，当然也会通知应用层

### onWindowUpdateRead
当收到一个window update时，表示对端调整了自己的接收窗口，这时需要通过encoder.flowController().incrementWindowSize()，通知encoder的flowController
变更所维护的发送window size，里面会触发streamByteDistributor.updateStreamableBytes


### onGoAwayRead
收到这个标示发送方准备shutdown了，这时接收方需要关闭那些比lastStreamId大的流，通知应用层做些操作

## outbound 数据处理

为了提高性能，netty http2将outbound的数据处理分为两个部分，一部分是由netty http2自己/应用程序触发的通过Http2FrameWriter的writeXXX接口进行write http2 frame的写提交过程，这过程只是提交需要进行的write操作，并不会真正的触发write系统调用。另外一部分则由Http2ConnectionHandler自己通过override `channelReadComplete`和`flush`驱动真正的写flush操作过程，这一过程将产生真正的write系统调用，将数据写往socket中。其中override`channelReadComplete`是为了能在一次eventloop轮完整read完、并处理完read得到的数据后进行一次flush写操作（一次eventloop轮中可能会调用`channelRead`多次，执行完后才会去执行一次`channelReadComplete`）。override `flush`操作是为了触发encoder进行write pending data。

关于channelReadComplete和channelRead的区别见[这篇回答](https://stackoverflow.com/questions/22354135/in-netty4-why-read-and-write-both-in-outboundhandler/22372777#22372777)


### part 1

### writeHeader


```plantumlcode
@startuml

-> DefaultHttp2ConnectionEncoder: writeHeaders
DefaultHttp2ConnectionEncoder -> Http2Connection: stream(streamId)
Http2Connection -> DefaultHttp2ConnectionEncoder : return stream (Http2Stream)

alt stream not exist
    DefaultHttp2ConnectionEncoder -> Http2Connection: local()
    Http2Connection -> DefaultHttp2ConnectionEncoder: return local Endpoint (DefaultEndpoint)
    DefaultHttp2ConnectionEncoder -> DefaultEndpoint: createStream(streamId)
    DefaultEndpoint -> DefaultStream : new DefaultStream(streamId, state)
    DefaultEndpoint -> DefaultEndpoint :  incrementExpectedStreamId
    DefaultEndpoint -> DefaultEndpoint : addStream(stream)
    DefaultEndpoint -> DefaultStream : stream.activate()
else stream.state() is RESERVED_LOCAL
    DefaultHttp2ConnectionEncoder -> DefaultStream : stream.open()
end

participant "Http2RemoteFlowController" as Http2RemoteFlowController #FF6600

DefaultHttp2ConnectionEncoder -> Http2Connection: remote().flowController()
Http2Connection -> DefaultHttp2ConnectionEncoder: return flowController

alt !endOfStream or !flowController.hasFlowControlled(stream)
    DefaultHttp2ConnectionEncoder -> DefaultHttp2FrameWriter : writeHeaders(...)
else trailing header
    DefaultHttp2ConnectionEncoder -> Http2RemoteFlowController : addFlowControlled(flowControlledHeaders)
end

@enduml
```

- 注意这里对于普通的header直接通过DefaultHttp2FrameWriter write给netty ctx，对于trailing header则需要经过Http2RemoteFlowController来保证和DATA帧的有序性。

### writeData
DATA帧需要经过flowControll，因此将调用通过```flowController().addFlowControlled(new FlowControlledData(...))``` 将数据交给RemoteflowController来处理。

```plantumlcode
@startuml

-> DefaultHttp2ConnectionEncoder: writeData
DefaultHttp2ConnectionEncoder -> Http2RemoteFlowController: addFlowControlled(...)
Http2RemoteFlowController -> WritabilityMonitor: enqueueFrame(state, frame)
WritabilityMonitor -> FlowState: enqueueFrame(frame)

alt current frame can be merge into last frame in the queue
    FlowState -> FlowState: pendingWriteQueue.peekLast().merge(frame)
    FlowState -> FlowState: incrementPendingByte(last.size() - previous last Size)
else 
    FlowState -> FlowState: enqueueFrameWithoutMerge(frame)
end
@enduml
```

这里值得注意的是Http2RemoteFlowController内部有个pendingWriteQueue。当有frame需要发送的时候，会先尝试将 FlowControlled 帧和pendingWriteQueue队尾的FlowControlled帧进行merge，以减少write次数

### writePriority
直接通过`frameWriter.writePriority`由DefaultHttp2FrameWriter写往netty ctx

### writeSettings
- 先将settings入队（outstandingLocalSettingsQueue）
- 再通过frameWriter.writeSetting将settings写给ctx
- 当收到对端的settingsAck时再应用settings，（见onSettingAckRead处理流程，从outstandingLocalSettingsQueue poll出setings）


### writeSettingsAck
- 正常情况下直接通过frameWriter.writeSettingAck将settings ack写给ctx

### writePing
- 直接通过frameWriter写给ctx

### writePushPromise
- 先在local端reserve一个push stream
- 然后通过frameWriter将push promise写给ctx

### writeWindowUpdate
- 不支持应用程序自己写window update帧

### writeRstStream
- 通过lifcyleManager也就是Http2ConnectionHandler本身去执行resetStream

### writeGoAway
- 和rstStream一样，通过lifcyleManager执行goAway

## part 2

```plantumlcode
@startuml
-> Http2ConnectionHandler : channelReadComplete()
Http2ConnectionHandler -> Http2ConnectionHandler: flush(ctx)
Http2ConnectionHandler -> Http2RemoteFlowController: writePendingBytes()
Http2RemoteFlowController -> WritabilityMonitor: writePendingBytes()
loop wribleable
    WritabilityMonitor -> StreamByteDistributor: distribute(writableBytes(), ...)
end
Http2ConnectionHandler -> Http2ConnectionHandler: ctx.flush()
@enduml
```

- writableBytes()返回当前connection最大的可写的字节数
- StreamByteDistributor.distribute(bytesToWrite) 分发所有流去实际的写最多bytesToWrite字节的数据。如果还有一些数据没能写出，返回true，否则返回false



#### StreamByteDistributor 接口定义
StreamByteDistributor的作用很明确，用于Remote flow controller，负责在并发的stream间分配字节。

内部定义了StreamState接口，用于给出当前stream的一些状态，比如：当前的stream有没有要写的frame（hasFrame方法），多少的pendingBytes（pendingBytes方法），当前的流控window大小 (windowSize方法)，窗口大小对于StreamByteDistributor来说很重要，分配的字节数不能超过当前的窗口大小。注意StreamByteDistributor定义这个接口是用来获取stream的内部状态，而非去操作或者修改这个状态，StreamState不是由StreamByteDistributor的实现类来实现，而是Http2 framework的其他组件来实现（这里其实是remote流控组件DefaultHttp2RemoteFlowController实现的）。

另外还定义了一个Writer接口，只有一个write(Http2Stream stream, int numBytes)方法，显然这个是在进行完分配计算后实际对stream进行写入时调用的方法。同样类似于StreamState接口，StreamByteDistributor定义这个接口的目的也是使用这个接口，而不是让子类来实现这个接口。

StreamByteDistributor本身直接定义了三个方法：

- updateStreamableBytes(StreamState state): 这个接口作用是用来告诉Distributor，StreamState标识的流的Streamable（可写的字节)发生变化了，以更新分配器内部的数据结构状态。调用的时机主要有：
    - 上层提交一个writeData要向流里写数据的时候
    - 流控窗口大小发生变更的时候（比如收到了对面的windowUpdate、完成了流上的数据实际写出）
    - 取消、发生错误等等时候
- updateDependencyTree(int childStreamId, int parentStreamId, short weight, boolean exclusive): 这个接口作用很明显，就是用于更新分配器内部stream的依赖树，调用的时机有两个：
    - 收到HEADER帧，标识对端需要创建一个流
    - 收到Priority帧，标识对端需要调整已存在的流的依赖关系或者优先级
- distribute(int maxBytes, Writer writer): 这个是分配器最核心的分配算法所在的接口，maxBytes是本轮最多允许分配的字节数, writer是实际写执行器。maxBytes取决于当前connection维度的流控窗口大小和当前netty channel最大可用的写字节数小值。
    - 这netty channel最大可用的写字节数有点意思：首先netty的channel有个ChannelOutboundBuffer（也就是netty自己的write buffer），这个buffer上定义了高水位线和低水位线，当buffer大小超过高水位线时，netty认为这个channel不能再写数据了，将isWritable置为false，并fire一个channelWritabilityChanged事件，这样应用程序可以停止写入；当buffer被flush到操作系统的socket buffer，使得大小低于低水位线后，buffer的isWritable将重新被置为true，此时应用程序就可以继续写入了。高低水位线的作用主要是防止OOM。这里的最大可用写字节数就是当前write buffer到高水位线的余量。但是这样存在个问题：假设连接质量不好或者对端处理很慢，导致write buffer比较满，可用字节数很少，导致每次实际分配的量很少，从而效率低下。因此netty对最大可用字节数做了最小值截断，使得最大可用字节数至少大于 max(netty channel的低水位, 32KB)。
    - distribute就一个，由channelReadComplete事件驱动


#### UniformStreamByteDistributor
算法策略：简单来说就是尽量均匀分配，内部有个队列，存放着可写的stream，在调用updateStreamableBytes时候入队stream，分配算法如下：
- 1）计算chunkSize = max(1KB, maxBytes / size)
- 2) 从队头取出stream，进行分配，实际分配大小为：min(chunkSize,  min(maxBytes, stream.streamableBytes)), 其中streamableBytes是min(stream.pendingBytes, stream.windowSize)
- 3) 遍历队列所有stream，重复2)，直到maxBytes为0，（当maxBytes为0导致分配提前结束时需要将stream重新放回队列）


注意：一个stream在分配后，想要再次得到分配的话需要再次调用updateStreamableBytes将stream入队，通过前面可以知道这个时机发生流控窗口的大小发生变更的时候（因为得到分配后写出了部分数据，发送窗口会减小）

#### WeightedFairQueueByteDistributor
参考了[nghttp2](https://nghttp2.org/)的[HTTP/2 priority implementation in nghttp2
](https://docs.google.com/presentation/d/1x3kWQncccrIL8OQmU1ERTJeq7F3rPwomqJ_wzcChvy8/edit#slide=id.p)的实现。顺便提一嘴，nghttp2是个很low-level的http2工具库，使用方有apache2、curl/libcurl、wireshark(HPACK only)...，作者也是http协议工作组

netty的WeightedFairQueueByteDistributor在这个[issue](https://github.com/netty/netty/issues/4462)被引入讨论。

具体算法思想和实现，受限于篇幅放到下一篇blog中介绍。


## 总结
Http2ConnectionHandler api我个人觉得定义的不太好，想要用起来需要深入了解很多内部的实现，这一情况在Http2FrameCodec、Http2MultiplexHandler引入后有所改观，但是目前http2 api也还没有完全stable下来，netty5的milestone里也计划将http2 api完全固定下来，以后将只会保留Http2MultiplexHandler api，这是个不错的想法。