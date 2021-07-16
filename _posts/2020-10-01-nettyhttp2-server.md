---
layout: post
title: "怎么基于netty构建一个HTTP2 server"
description: ""
category: 
tags: []
---

# 怎么基于netty构建一个HTTP2 server

## 配置h2 server

h2 server是基于TLS的ALPN的扩展来start http2的，因此需要先为pipeline配置上ssl handler，然后监听ssl 握手完成事件，得到ALPN的protocol，再根据protocol来配置pipeline。


netty里已经提供好一个`ApplicationProtocolNegotiationHandler`基类，通过实现其`configurePipeline`方法来根据ALPN扩展里的protocol来配置相应的pipeline

<!--more-->

```
    @Override
    protected void configurePipeline(ChannelHandlerContext ctx, String protocol) throws Exception {
        if (ApplicationProtocolNames.HTTP_2.equals(protocol)) {
            ctx.pipeline().addLast(new HelloWorldHttp2HandlerBuilder().build());
            return;
        }

        if (ApplicationProtocolNames.HTTP_1_1.equals(protocol)) {
            ctx.pipeline().addLast(new HttpServerCodec(),
                                   new HttpObjectAggregator(MAX_CONTENT_LENGTH),
                                   new HelloWorldHttp1Handler("ALPN Negotiation"));
            return;
        }

        throw new IllegalStateException("unknown protocol: " + protocol);
    }
```


HelloWorldHttp2HandlerBuilder().build()部分就是创建http2相关的handler， 后面还会详细介绍。

## 配置h2c server (client没有server的是否是http2的先验知识)

h2c server跑在原始的tcp connnection上，需要先通过http1的协议upgrade机制来升级协议，因此pipeline里首先需要add的是http1相关的handler，主要有两个
- HttpServerCodec
- HttpServerUpgradeHandler

### HttpServerCodec

上面的代码中，HttpServerCodec是netty-http1的协议codec，他是一个duplex型的handler，既是一个decoder，又是一个encoder。由于其完全属于http1协议的内容，这里就不多叙述了


### HttpServerUpgradeHandler
HttpServerUpgradeHandler是nettp-http1中专门处理协议upgrade的handler，他定义了两个接口，SourceCodec和UpgradeCodec。

```
    //http1协议的codec（HttpServerCodec），upgradeFrom里需要把自己从pipeline里remove掉
    public interface SourceCodec {
        void upgradeFrom(ChannelHandlerContext ctx);
    }
    
    
    public interface UpgradeCodec {
        //定义新协议需要的headers
        Collection<CharSequence> requiredUpgradeHeaders();
        
        //用来给新协议检查、处理upgradeRequest
        boolean prepareUpgradeResponse(ChannelHandlerContext ctx, FullHttpRequest upgradeRequest,
                                    HttpHeaders upgradeHeaders);
        //配置pipeline采用新协议
        void upgradeTo(ChannelHandlerContext ctx, FullHttpRequest upgradeRequest);
    }
```

upgrade过程就是：（HttpServerUpgradeHandler.upgrade）
- 校验headers里包含UpgradeCodec定义的所有的requiredUpgradeHeaders
- 创建一个upgrade protocol response, 即101 switch protocol
- 调用UpgradeCodec的prepareUpgradeResponse, 给新协议的codec一个检查、处理upgradeRequest的机会，并且可以根据request选择是否中断upgrade过程
- 就地写回upgrade protocol response给client。注意必要要这个时候写回upgrade protocol response，因为马上原来的http1 codec就要被remove了，connection马上是按照新协议来通信，所以需要趁着http1的codec还在的时候写回upgrade protocol response。
- 调用SourceCodec（也就是http1的HttpServerCodec）的upgradeFrom方法，让其从pipeline中删掉自己
- 调用UpgradeCodec的upgradeTo方法，配置新协议的pipeline
- 触发一个userEvent，通知新的pipeline upgrade成功


下面是Http2 upgrade相关的，主要有
### Http2ServerUpgradeCodec

#### 初始化

Http2ServerUpgradeCodec的初始化方法里最重要的是传入一个Http2ConnectionHandler对象，它将是http2协议最主要的handler。在示例代码中通过
`new Http2ServerUpgradeCodec(new HelloWorldHttp2HandlerBuilder().build());` 来构造Http2ServerUpgradeCodec对象，`new HelloWorldHttp2HandlerBuilder().build()` 在上一节中已经见过了, 他继承自netty http2最重要的handler —— `Http2ConnectionHandler`，后面详细介绍。

除此之外，在Http2ServerUpgradeCodec的构造方法中还创建了一个Http2FrameReader接口的实例对象(DefaultHttp2FrameReader), Htttp2FrameReader其实就是http2协议的decoder，但是并不是一个netty handler，而是一个独立的decoder，原因一会再说。

#### 实现requiredUpgradeHeaders
这里只有一个header是必须的： 也就是HTTP2-Settings

#### 实现prepareUpgradeResponse
主要是从http1的FullHttpRequest中拿到HTTP2-Settings header，按照http2协议他是HTTP/2 SETTINGS的payload进行base64url的结果。因此需要先base64 decode，然后用前面提到的Http2FrameReader，也就是http2协议的decoder去decode出Http2Settings帧来。这也是什么从netty handler中独立出Http2 decoder的原因之一。

这里面有个小细节，就是Http2FrameReader是从http2 frame的header和payload构成的二进制decode出完整的http2 frame，但是这里的HTTP2-Settings 头只是Http2Settings帧的 payload，因此为了代码复用需要先从payload创建完整的header+payload二进制，然后交给Http2FrameReader去decode。

```
    private Http2Settings decodeSettingsHeader(ChannelHandlerContext ctx, CharSequence settingsHeader)
            throws Http2Exception {
        ByteBuf header = ByteBufUtil.encodeString(ctx.alloc(), CharBuffer.wrap(settingsHeader), CharsetUtil.UTF_8);
        try {
            // Decode the SETTINGS payload.
            ByteBuf payload = Base64.decode(header, URL_SAFE);

            // Create an HTTP/2 frame for the settings.
            ByteBuf frame = createSettingsFrame(ctx, payload);

            // Decode the SETTINGS frame and return the settings object.
            return decodeSettings(ctx, frame);
        } finally {
            header.release();
        }
    }

    /**
     * Decodes the settings frame and returns the settings.
     */
    private Http2Settings decodeSettings(ChannelHandlerContext ctx, ByteBuf frame) throws Http2Exception {
        try {
            final Http2Settings decodedSettings = new Http2Settings();
            frameReader.readFrame(ctx, frame, new Http2FrameAdapter() {
                @Override
                public void onSettingsRead(ChannelHandlerContext ctx, Http2Settings settings) {
                    decodedSettings.copyFrom(settings);
                }
            });
            return decodedSettings;
        } finally {
            frame.release();
        }
    }
```

#### 实现upgradeTo
就是将Http2ConnectionHandler 加到netty pipeline中，Http2ConnectionHandler的handlerAdded方法中做了不少初始化的事情，另外就是初始化了一个PrefaceDecoder，这会使得server开始发送connection preface给client。然后调用Http2ConnectionHandler.onHttpServerUpgrade把Http2Settings frame传给Http2ConnectionHandler，以便做一些初始化事情





## 配置h2c server (client知道server是http2)
有些client可能知道server 提供http2服务这一先验知识，因此他们可以直接和server进行http2通信，省去了HTTP1 upgrade这一交互过程。按照协议双方直接开始交互CONNECTION PREFACE (一串特殊字符串后跟着Http2Settings帧)，因此再server可以根据连接上的开始字节是否是CONNECTION PREFACE来配置pipeline，这一点netty提供了
CleartextHttp2ServerUpgradeHandler，封装了这一个过程。

```
private void configureClearText(SocketChannel ch) {
        final ChannelPipeline p = ch.pipeline();
        final HttpServerCodec sourceCodec = new HttpServerCodec();
        final HttpServerUpgradeHandler upgradeHandler = new HttpServerUpgradeHandler(sourceCodec, upgradeCodecFactory);
        final CleartextHttp2ServerUpgradeHandler cleartextHttp2ServerUpgradeHandler =
                new CleartextHttp2ServerUpgradeHandler(sourceCodec, upgradeHandler,
                                                       new HelloWorldHttp2HandlerBuilder().build());

        p.addLast(cleartextHttp2ServerUpgradeHandler);
        ...
}
```


## Http2ConnectionHandler

虽然说Http2ConnectionHandler是netty http2中最重要的handler，但是实现的非常原始，使用起来也很难用，不太符合直觉。

设计上，他可以说主要是Inbound handler，继承自ByteToMessageDecoder，通过内部decoder来对http2协议进行decode，decoder内部再委托给使用者自己实现的Http2FrameListener对收到的http2消息进行业务处理。这里和netty中常见的decoder decode出协议封装成对象消息交给pipeline下游业务handler的做法不太一样。另外，他在实现上又实现了ChannelOutboundHandler接口，看上去主要是为了实现flush方法，并不是作为一个encoder handler。如果使用方想去向连接上响应http2的response还需要通过他的encoder()方法拿到一个Http2FrameWriter对象，通过Http2FrameWriter的各种writeXXX接口直接write对应的http2消息/帧，可以说非常原始了。

### Http2FrameCodec

可能开发者也意识Http2ConnectionHandler的原始和难用，以及缺少对http2 frame的封装，因此又引入了`Http2StreamFrame`接口对http2 frame进行封装，以及`Http2FrameCodec`作为http2 frame的codec继承并封装`Http2ConnectionHandler`，使得`Http2ConnectionHandler`的使用更符合直觉。如此一来配置一个http2的server pipeline变得开箱即用了：


- h2

```
ctx.pipeline().addLast(Http2FrameCodecBuilder.forServer().build(), new HelloWorldHttp2Handler());

```

- h2c

```
    final ChannelPipeline p = ch.pipeline();
    final HttpServerCodec sourceCodec = new HttpServerCodec();
    p.addLast(sourceCodec);
    //http1的server upgrade handler， 指定upgradeCodecFactory
    p.addLast(new HttpServerUpgradeHandler(sourceCodec, upgradeCodecFactory));

    
    ...
    
    private static final UpgradeCodecFactory upgradeCodecFactory = new UpgradeCodecFactory() {
        @Override
        public UpgradeCodec newUpgradeCodec(CharSequence protocol) {
            if (AsciiString.contentEquals(Http2CodecUtil.HTTP_UPGRADE_PROTOCOL_NAME, protocol)) {
                return new Http2ServerUpgradeCodec(
                        Http2FrameCodecBuilder.forServer().build(), new HelloWorldHttp2Handler());
            } else {
                return null;
            }
        }
    };

```

上面`Http2FrameCodecBuilder.forServer().build()`就是创建了一个开箱即用的Http2FrameCodec，`HelloWorldHttp2Handler`则完全是业务handler，节选了部分代码：

```
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof Http2HeadersFrame) {
            onHeadersRead(ctx, (Http2HeadersFrame) msg);
        } else if (msg instanceof Http2DataFrame) {
            onDataRead(ctx, (Http2DataFrame) msg);
        } else {
            super.channelRead(ctx, msg);
        }
    }

    private static void sendResponse(ChannelHandlerContext ctx, Http2FrameStream stream, ByteBuf payload) {
        // Send a frame for the response status
        Http2Headers headers = new DefaultHttp2Headers().status(OK.codeAsText());
        ctx.write(new DefaultHttp2HeadersFrame(headers).stream(stream));  //注意这里需要指定frame所生效的stream
        ctx.write(new DefaultHttp2DataFrame(payload, true).stream(stream));
    }
```

回到`Http2FrameCodec`本身，他作为http2 frame的codec，负责对当前存活着的stream上frame进行decode和encode, 和下游handler之间通过`Http2StreamFrame`进行交互。

`Http2StreamFrame` 内部attach了一个`Http2FrameStream`对象唯一标识frame所在的stream。所有从channel上读取到的`Http2StreamFrame`都有其对应的`Http2FrameStream`属性，另外，所有写给`Http2FrameCodec`的`Http2StreamFrame`也都需要在应用代码里指定`Http2FrameStream`属性，通过`Http2StreamFrame.stream(Http2FrameStream)`设置frame所生效的stream。

Http2FrameCodec还提供了创建一个outbound stream的api：
```
 final Http2Stream2 stream = handler.newStream();
 ctx.write(headersFrame.stream(stream));
```



### Http2MultiplexHandler

这个类主要是对http2 stream复用这个行为进行了封装，当连接上有新的stream创建时，为其创建一个新的`Http2MultiplexHandlerStreamChannel`，并注册到eventLoop中，并对应用代码的handler屏蔽或者转换一些原有连接上frame消息（主要是控制frame），使得应用handler面向新的`Http2MultiplexHandlerStreamChannel`编程，只需要关注子channel上的事件。

使用上需要和Http2FrameCodec一起使用，如：
```
        if (ApplicationProtocolNames.HTTP_2.equals(protocol)) {
            ctx.pipeline().addLast(Http2FrameCodecBuilder.forServer().build());
            ctx.pipeline().addLast(new Http2MultiplexHandler(new HelloWorldHttp2Handler()));
            return;
        }
```
Http2MultiplexHandler作为Http2FrameCodec的下游handler，注意应用程序handler——HelloWorldHttp2Handler作为构造参数传给了Http2MultiplexHandler，此外HelloWorldHttp2Handler区别于直接使用Http2FrameCodec时，在写回响应时候更简单些：

```
    private static void sendResponse(ChannelHandlerContext ctx, ByteBuf payload) {
        // Send a frame for the response status
        Http2Headers headers = new DefaultHttp2Headers().status(OK.codeAsText());
        ctx.write(new DefaultHttp2HeadersFrame(headers));  //这里不需要再给DefaultHttp2HeadersFrame指定所生效的stream了，因为当前channel就是所生效的stream。
        ctx.write(new DefaultHttp2DataFrame(payload, true));
    }
```

