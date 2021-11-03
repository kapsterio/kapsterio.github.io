---
layout: post
title: "tls1.2 handshake"
description: ""
category: 
tags: []
---

## Overview

这篇blog是关于tls1.2协议handshake部分的介绍和分析，TLS握手协议作用主要有以下几点：
- 协商出双方协议的版本、加密算法套件
- 认证对方身份
- 协商出一个用于后续加密数据的密钥


一个完整的TLS握手过程如下图所示：

![tls handshake](/public/fig/full_handshake.png)


<!--more-->

第一轮握手包括以下几个步骤：
- ClientHello和ServerHello作用是为了协商协议版本号、session ID、加密套件、压缩算法等，另外双方还生成并交换了两个随机数，ClientHello.random和ServerHello.random
- 紧接着server将自己的证书通过Certificate消息发过去
- server端此时可能需要通过ServerKeyExchange消息将密钥交换所需的server part数据发过去(比如EDH中动态DH公钥和签名)
- 如果server端配置了要验证client身份，server还会发个CertificateRequest消息
- hello阶段的最后server发送ServerHelloDone消息作为结束。


第二轮握手则包括下面这些：
- Client端如果收到server的CertificateRequest消息，则需要将自己的证书通过Certificate消息发过去。
- Client端此时验证完server证书了，需要进行密钥交换，发送一个ClientKeyExchange消息过去，同样消息内容取决于协商的密钥交换算法，至此对于Client来说，可以让密钥交换算法工作计算得到共享密钥了。
- 如果Server端要求Client提供证书，Client端除了发送Certificate之外，这时还会发送一个CertificateVerify消息，里面带有对目前连接上所有消息的签名，以表明自己拥有证书。这点值得说明下，证书里面包含公钥，都是公开的，攻击者也能拿到。如果server需要client端证明身份的话不仅需要client端将证书发过去，毕竟攻击者完全可以拿到这个合法的证书，假装是自己的发给server端。还需要client端通过这个CertificateVerify消息对连接上目前所有发过的握手消息进行签名(这里当然也包括server hello中的server random， 因此也就避免了重放攻击)，签名涉及到私钥，client端如果能正确签名的话说明拥有私钥，server验证签名后就能确信对端拥有证书了。
- 此时，Client端将发送一个ChangeCipherSpec，以表明后续将采用加密通信了，这个消息在tls1.3中去掉了。
- Client最后会发送Finished握手完成消息，Finished消息是加密消息了，第三方无法看到内容，作用也是为了验证握手阶段的所有消息没有被篡改。
- Server端收到Client端消息后，可选的验证身份，密钥交换算法工作计算得到共享密钥。然后给Client回复ChangeCipherSpec消息，标识后续通信也开始采用加密协议了。
- 最后Server端发送Finished消息，握手结束。Client端收到Server 的Finished消息后，开始应用数据传输。


## Handshake Protocal frame
![handshake frame](/public/fig/handshake_frame.png)


### Client Hello
![client hello](/public/fig/client_hello.png)
- client version： 用于协商版本，client将自己支持的最高version发给server端，server端收到后将彼此支持的最高version返回给client。
- client random 
- SessionID
- CipherSuite list
- Compression method list
- Extension block是tls协议的扩展点，客户端将自己支持的扩展以及对应的数据通过这个字段告诉server。


### Server Hello
![server hello](/public/fig/server_hello.png)


#### extensions
常见的扩展有：
- ALPN (Application Layer Protocol Negotiation)
- SNI (Server Name Indication)
- signature_algorithms： 这个扩展用于告知server端客户端所支持的签名算法，服务端会根据它来选择证书，没有这个扩展之前，比如一个RSA公钥证书里签名部分只能由于RSA签名算法签署，有了它之后证书里RSA公钥完全可以其他签名算法签署（比如ECDSA），也就是说客户端会通过ECDSA算法来验证证书，证书里的RSA公钥则可能用于验证ServerKeyExchange消息中的DH共钥。




### Server Certificate

证书不在tls协议范围内，更多内容可以参考[everything-pki](https://smallstep.com/blog/everything-pki/)

### Server key exchange

- 不是所有握手过程都需要Server key exchange消息，只有ciper suite中密钥交换算法是Ephemeral Diffie-Hellman时候才需要它，通过这个消息将短时server DH 公钥传给客户端。
- 除了DH公钥外，还需要将该公钥 + client & server random的签名也传过去，以证明自己持有这个公钥。这里也是签名算法server端除了证书之外唯一应用的地方。


### Certificate Request
当server需要认证客户端身份时，server需要发送这个消息，具体字段其实也不太重要


### Server Hello Done

第一轮握手以Server发送Server Hello done作为结束，接下来由client继续表演。该消息没有消息体。

### Client Certificate

当server要求认证客户端身份时，client端首先需要发送client证书过来，和Server Certificate消息类似。
- 问题：client没有域名，server怎么验证？

### Client Key Exchange Message
这个消息的内容取决于协商的密钥交换算法
- 如果是RSA-based密钥交换：那么该消息内容是RSA加密后的PreMasterSecret, PreMasterSecret是client hello version + 46 bytes random构成，version为了防止中间人的版本回退攻击 
- 如果是DH-based密钥交换： 那么该消息内容则是client端的DH公钥，PreMasterSecret则由各端将对端的DH公钥和自己的DH私钥进行DH计算得到。
- 该消息结束后双端都已经能够计算master secret了，计算方法如下：
```
master_secret = PRF(pre_master_secret, "master secret",                          ClientHello.random + ServerHello.random)                     [0..47];
```

注意这里master secret的计算依赖pre_master_secret、client random和server random，

### Certificate Verify
这个消息作用前文也解释了，当Server端要求Client进行身份认证时，Client端除了发送Certificate之外，这时还会发送一个CertificateVerify消息，里面带有对目前连接上所有消息的签名，以表明自己拥有证书。需要注意的是该消息的发送时机，是在client key exchange后，因为签名消息里也要包含client key exchange消息。

- 思考：为什么server端不需要类似这样的一个verify消息来证书自己拥有server证书。

### [ChangeCipherSpec]
ChangeCipherSpec 作用是为了告知对端接下来的消息是加密后的密文了。其本身并不属于handshake消息类型，在tls1.3中也被移除了。

### Finished
一旦一端完成了密钥交换，在发完[ChangeCipherSpec]消息后，就会发送Finished消息，作用了为了让对端验证身份认证和密钥交换过程成功进行，其实就是对所有handshake消息采用master_secret计算一个MAC，然后和application data一样使用client write key加密发给对端。

需要特别说明的时从Finished消息开始，消息内容才是加密后的内容，finished消息之前的所有消息对中间人都是可见的，这就给MITM以机会篡改握手消息进行降级攻击，使得client和server协商出一个并非本意的弱密钥，MITM破解后只要对Finished消息的hash进行重新计算，就可以在client和server无感知的情况下控制整个会话，这也是[FREAK attack](https://en.wikipedia.org/wiki/FREAK)的攻击原理。

![freak](/public/fig/freak.png)


## Session Resumption

根据[cloudflare的数据](https://blog.cloudflare.com/introducing-0-rtt/)，60%的tls连接是全新连接，剩下40%连接都是从之前连接恢复出的连接，意味有这40%的网页访问请求行为在短时间内还好重复发生。Session resumption机制就是tls用于从之前创建过的连接恢复出连接的机制，从避免身份认证、密钥交换等一些列代价高的操作。tls1.2中有两种用于恢复连接的机制。

###  Session Identifiers

session ID机制要求server端生成并维护一个session id到session状态信息的cache，并在server hello信息里将session id返回给客户端，客户端下次握手时在client hello里带上这个session id。server那到这个session id后从cache中将上次session的状态恢复出来，直接完成握手。因此一个带session id的握手将会是如下图所示：

![session id](/public/fig/session_id.png)

可以看出session id可以使得握手过程减少一个rtt，代价就是这个session cache，为了解决这个问题tls在[RFC5077](https://datatracker.ietf.org/doc/html/rfc5077)中引入session ticket机制

### Session ticket
思想很简单，把存储session信息的代价分散到客户端就好了，从而避免在server端维护session cache的代价。放客户端肯定不能明文存储，所以就加密后再给客户端。

具体来说： 第一次进行tls握手时候，支持session ticket的客户端在client hello里带上空的session ticket 扩展，server端回复的server hello里也会带上空的session ticket扩展，双方都知道彼此认识这个扩展。 然后server端完成密钥交换后，发送一个NewSessionTicket握手消息给client端，里面包含server端加密后的session信息（ticket）以及ticket的过期时间。
![ticket generation](/public/fig/ticket_generation.png)

client端收到NewSessionTicket后将当前session信息和server给的session ticket相关联并保存下来。后续想要恢复session时，client在client hello的session ticket扩展中带上之前保存的ticket，server收到后如果能够正确解析ticket，则可以从中恢复出之前的server端session信息，至此两端都恢复出之前的session状态，完成握手。
![ticket used](/public/fig/ticket_used.png)


当然ticket完全可能被攻击者所截获，攻击者如果拿着client的ticket去尝试和server恢复session时由于没有master_secret，因此失败在Finished消息阶段。


关于这个session状态，协议中并没有强制规定包括哪些，但是协议给了建议要保存的一些状态，如下所示，其中最主要的就是master_secret和timestamp。
```
struct {           
    ProtocolVersion protocol_version;           
    CipherSuite cipher_suite;           
    CompressionMethod compression_method;           
    opaque master_secret[48];           
    ClientIdentity client_identity;           
    uint32 timestamp;       
} StatePlaintext;
```


由于ticket对client是透明的，因此协议并也没有强制规定ticket应该怎么构成，只做了下格式和密码学保护上的实现建议，如下：
```
struct {           
    opaque key_name[16];           
    opaque iv[16];           
    opaque encrypted_state<0..2^16-1>;           
    opaque mac[32];       
} ticket;
```
首先状态信息需要安全加密并用mac保护，因此涉及到两个key，一个key用来AES加密state，一个key用来生成mac，key_name用来告诉server 这两个key是什么，协议还建议server启动的时候去生成随机的key_name以及对应的keys。



## tls false start

[RFC7918](https://datatracker.ietf.org/doc/html/rfc7918)

## tls后向兼容

前面有提到tls的版本协商机制，这里再展开多说一点：设计上很简单，举个例子，client支持tls1.2，server只支持tls1.0，server应该将告诉client用tls1.0通信，非常简单的版本协商机制。问题就出在一些不合作的server，发现不支持client的版本后，直接给断掉连接了，让浏览器怎么办？在2015/16年之前，浏览器为了兼容这些不老实的server纷纷采用的版本fallback机制，即在server中止握手后尝试一个更低的版本，直至连接建立成功，这么做一方面损坏了用户体验（重试好几次可能），另一方面给版本回退攻击开了口子，攻击者可以阻断连接，给client造成server只支持低版本tls的假象，然后利用低版本tls的漏洞破解tls session。

这个版本协商机制要求server实现要能够正确的做到后向兼容，应对未来的情况，通常来说是很难搞对，因为缺少反馈机制，搞错了在当时也没人知道，只有等到几年后协议升级, 新的client出现问题才暴露，这时候错误的server已经在市场是广泛存在了，升级还需要花个几年才能纠正错误行为。因此tls这个简单的版本协商机制，看起来设计上简单，其实是一种典型的protocol design anti-pattern。

如果一种机制虽然设计上是灵活的，但是实际上从来没变过，那么肯定会有人会认为他是不变的，而网络协议能够正确work涉及整个链路上包括client、中间件、server等所有参与方的协同工作，一旦有环节这么干了，就导致了protocol ossification，好的设计要能规避掉实现方出错的可能。我们会在tls1.3内容里再次看到tls1.2设计导致的协议骨化问题，以及tls1.3的解决方案。


关于tls后向兼容的话题可以参考[cloudflare的这篇blog](https://blog.cloudflare.com/why-tls-1-3-isnt-in-browsers-yet/)