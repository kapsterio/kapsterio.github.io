---
layout: post
title: "怎么基于netty构建一个HTTP2 client"
description: ""
category: 
tags: []
---


# 怎么基于netty创建一个HTTP2 client

## 配置h2的client pipeline
```
    private void configureSsl(SocketChannel ch) {
        ChannelPipeline pipeline = ch.pipeline();
        // Specify Host in SSLContext New Handler to add TLS SNI Extension
        pipeline.addLast(sslCtx.newHandler(ch.alloc(), Http2Client.HOST, Http2Client.PORT));
        // We must wait for the handshake to finish and the protocol to be negotiated before configuring
        // the HTTP/2 components of the pipeline.
        pipeline.addLast(new ApplicationProtocolNegotiationHandler("") {
            @Override
            protected void configurePipeline(ChannelHandlerContext ctx, String protocol) {
                if (ApplicationProtocolNames.HTTP_2.equals(protocol)) {
                    ChannelPipeline p = ctx.pipeline();
                    p.addLast(connectionHandler);
                    configureEndOfPipeline(p);
                    return;
                }
                ctx.close();
                throw new IllegalStateException("unknown protocol: " + protocol);
            }
        });
    }
```
可以看出还是熟悉的味道，基于ALPN扩展协商，然后将connectionHandler也就是Http2ConnectionHandler加入pipeline中。

## 配置h2c client pipeline

```
    private void configureClearText(SocketChannel ch) {
        HttpClientCodec sourceCodec = new HttpClientCodec();
        Http2ClientUpgradeCodec upgradeCodec = new Http2ClientUpgradeCodec(connectionHandler);
        HttpClientUpgradeHandler upgradeHandler = new HttpClientUpgradeHandler(sourceCodec, upgradeCodec, 65536);

        ch.pipeline().addLast(sourceCodec,
                              upgradeHandler,
                              new UpgradeRequestHandler());
    }
```
基本和之前配置h2c server pipeline对应：
- HttpClientCodec 对应 HttpServerCodec， 都属于http1协议的codec
- HttpClientUpgradeHandler 对应HttpServerUpgradeHandler，也属于http1，作用是处理http1 client端upgrade过程。
- Http2ClientUpgradeCodec 对应 Http2ServerUpgradeCodec， 作用也是在upgradeTo方法被调用时将http2的connectionHandler加到pipeline中。
- UpgradeRequestHandler 有点特殊，但是作用也很简单，就是在连接建立后立即发起一个最简单的http request用于触发upgrade过程。


### HttpClientUpgradeHandler

这里多写些关于HttpClientUpgradeHandler的内容，作用前面也提到了负责client端HTTP upgrade过程，他的工作分为两个阶段，首先client发起upgrade http请求时他负责向请求里写入合适的headers（这些header是要升级的新协议里定义的）。其次，收到upgrade请求的响应(101 switching protocol)后他要负责将pipeline重新配置以便开始进行新协议通信。因此他既是一个inbound handler也是一个outbound handler。

和HttpServerUpgradeHandler一样，他也定义了两个接口
```
    public interface SourceCodec {

        //暂不明确这是干嘛的, 看代码作用是让http1的encoder失效。但是为什么保留decoder呢
        void prepareUpgradeFrom(ChannelHandlerContext ctx);

        //用于从pipeline中删掉自己
        void upgradeFrom(ChannelHandlerContext ctx);
    }

    
    public interface UpgradeCodec {
        //定义新协议的名字，作为header中`UPGRADE`的值
        CharSequence protocol();
        
        //用于设置一些新协议相关的header，同时返回新协议所设置的headers name集合
        Collection<CharSequence> setUpgradeHeaders(ChannelHandlerContext ctx, HttpRequest upgradeRequest);

        //用于新协议配置pipeline
        void upgradeTo(ChannelHandlerContext ctx, FullHttpResponse upgradeResponse) throws Exception;
    }
```

在收到101 switching protocol响应后的处理流程是：(这个过程的实现要稍微难理解一些)

- 调用sourceCodec的prepareUpgradeFrom, disable掉sourceCodec的encoder，为什么不能直接删除sourceCodec呢？
- 调用upgradeCodec的upgradeTo将pipeline配置为新协议的，（对于http2而言connection preface也会在这个时候发送，所以http1的encoder必须要disable掉）
- 向ctx去fire一个userEvent  (UPGRADE_SUCCESSFUL)
- 最后调用sourceCodec的upgradeFrom方法将sourceCodec从pipeline中删掉，这一过程会导致decoder的handlerRemoved方法被调用，其内部buffer中可能会已经有来自server的connection preface数据了，所以handlerRemoved将会触发一次channelRead，因此只能放在upgradeCodec配置好之后进行。(Trustin Lee也在commit message里解释了为什么引入prepareUpgradeFrom以及souceCodec.upgradeFrom放到upgradeCodec.upradeTo后面)
- 最后把HttpClientUpgradeHandler本身也从pipeline中删除


## 配置h2c client pipeline （当知道server提供http2服务）

```
    protected void initChannel(Channel ch) throws Exception {
        // ensure that our 'trust all' SSL handler is the first in the pipeline if SSL is enabled.
        if (sslCtx != null) {
            ch.pipeline().addFirst(sslCtx.newHandler(ch.alloc()));
        }

        final Http2FrameCodec http2FrameCodec = Http2FrameCodecBuilder.forClient()
            .initialSettings(Http2Settings.defaultSettings()) // this is the default, but shows it can be changed.
            .build();
        ch.pipeline().addLast(http2FrameCodec);
        ch.pipeline().addLast(new Http2MultiplexHandler(new SimpleChannelInboundHandler() {
            @Override
            protected void channelRead0(ChannelHandlerContext ctx, Object msg) {
                // NOOP (this is the handler for 'inbound' streams, which is not relevant in this example)
            }
        }));
    }
```
通过Http2FrameCodecBuilder.forClient build出一个针对client的Http2FrameCodec，这里同时也使用了Http2MultiplexHandler用于封装stream。
由于Http2MultiplexHandler的构造参数的inboundStreamHandler仅对那些由Http2MultiplexHandler创建的streamChannel生效，也就是所谓的inbound stream生效 (对client而言inbound stream就是server push创建的stream)。所以这里的SimpleChannelInboundHandler啥也不干。

那问题来了，client怎么创建一个outbound stream呢，netty提供了Http2StreamChannelBootstrap:

```
            final Http2ClientStreamFrameResponseHandler streamFrameResponseHandler =
                    new Http2ClientStreamFrameResponseHandler();

            final Http2StreamChannelBootstrap streamChannelBootstrap = new Http2StreamChannelBootstrap(channel);
            final Http2StreamChannel streamChannel = streamChannelBootstrap.open().syncUninterruptibly().getNow();
            streamChannel.pipeline().addLast(streamFrameResponseHandler);

            // Send request (a HTTP/2 HEADERS frame - with ':method = GET' in this case)
            final DefaultHttp2Headers headers = new DefaultHttp2Headers();
            headers.method("GET");
            headers.path(PATH);
            headers.scheme(SSL? "https" : "http");
            final Http2HeadersFrame headersFrame = new DefaultHttp2HeadersFrame(headers, true);
            streamChannel.writeAndFlush(headersFrame);
```

通过Http2StreamChannelBootstrap， 可以通过open方法像创建channel一样创建一个Http2StreamChannel作为client发起的stream；还可以在这个streamChannel的pipeline上add handler，这个handler将只会处理这个streamChannel上的读写事件。