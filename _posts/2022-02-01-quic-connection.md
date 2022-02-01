---
layout: post
title: "QUIC Connection"
description: ""
category: 
tags: []
---

## 0. Overview

QUIC协议的RFC大概是目前为止我个人读过的最复杂的RFC文档，光是RFC文档都分为了4篇，分别是:

- [RFC 8999]([https://www.rfc-editor.org/rfc/rfc8999.html](https://www.rfc-editor.org/rfc/rfc8999.html))规定了QUIC所有版本都将保持不变的一些设计
- [RFC 9000]([https://www.rfc-editor.org/rfc/rfc9000.html](https://www.rfc-editor.org/rfc/rfc9000.html))是协议的框架以及除了连接建立外的协议核心部分
- [RFC 9001]([https://www.rfc-editor.org/rfc/rfc9001.html](https://www.rfc-editor.org/rfc/rfc9001.html)) 是QUIC整合TLS1.3进行密码学握手建立安全的QUIC连接部分
- [RFC 9002]([https://www.rfc-editor.org/rfc/rfc9002.html](https://www.rfc-editor.org/rfc/rfc9002.html)) 则是关于QUIC丢包检测拥塞控制部分

RFC 9001在之前的[blog]([http://kapsterio.github.io/test/2021/12/28/quic-handshake.html](http://kapsterio.github.io/test/2021/12/28/quic-handshake.html))中已经做了解读，这篇是继QUIC handshake之后QUIC连接相关的协议解读，主要内容来自于RFC 9000中的第5-第10章节，并按照连接建立、连接迁移、连接终止顺序做了一些内容上的重新组织。另外RFC9000包含的信息量非常大，对很多QUIC设计和行为规定也没有做必要解释，这也造成了阅读体验上枯燥和难懂，因此这篇blog 里尽量对一些我个人认为比较难以理解的点做些motivation上的补充说明。看完并理解这部分内容的话，读者就可以理解QUIC的整个连接建立过程和连接终止过程的工作细节。

PS: QUIC Connection中有很多地方设计上参考了DTLS，了解下DTLS是怎么做的个人觉得有助于理解QUIC，这部分具体可以参考[DTL1.2](http://kapsterio.github.io/test/2022/01/01/dtls12.html)和[DTLS1.3](http://kapsterio.github.io/test/2022/01/09/dtls13.html)。

<!--more-->

## 1. QUIC Connection ID

### Connection ID协商

QUIC的Connection ID的协商机制在基本设计上和DTLS的类似，都是两端独立选择两个方向上的CID，用来告诉对端希望对端后面用这个CID来和自己通信。

不同的是QUIC有两个CID，Destination CID和Source CID：

- Destination CID: (简称DCID)就是前面提到的 CID，是由对端选择或者生成的，本端使用。
- Source CID: (简称为SCID) 用于本段告诉对端自己选择的CID，说白了SCID是用来协商CID的字段。

这里简单解释下，在之前的DTLS协议中CID 是以TLS 扩展的形式存在的，CID的协商需要双方在 ClientHello/ServerHello消息中带上ConnectionID扩展，独立选择各自方向上的CID。在QUIC中 CID是完全的协议原住民，不依赖任何扩展，所以CID协商也是随着QUIC packet long header的交换来进行的，因此long header中才有SCID（用于交换本段CID）和DCID （使用对端CID），short header中则只有DCID。

所以这么看来，在client 的initial packet (使用long header)中，DCID是没有必要存在的，只需要SCID就行了，因为此时client还没有server 的CID。但是我们知道协议QUIC的initial secrets 的生成又依赖DCID，因此QUIC协议中规定client在发送initial packet时候，DCID要设置为一个不可预测的值，并且在收到server的packet前，所有Client的packet 需要采用一致的DCID值（这里主要指的是client initial packet、0-rtt packet）。收到server的initial packet或者是retry packet后，client后续的DCID将采用server packet里指定的SCID。

还有一点：SCID可以为空，意味着这本端选择空的CID给对端，对端发送的packet里的DCID将为空，packet 的达到本端后的路由将不依赖于CID。一般client端都会选择空的SCID。一旦在初始协商CID时将SCID置空，后面整个connection的生命周期内本端都不能再生产CID。


### Connection ID更新

在DTLS1.3中，任意一端可以通过**RequestConnectionId**消息发起ConnectionID的更新请求，来让对端进行CID的生产。对端收到后生产新的CIDs并回应一个**NewConnectionId**消息。可以称之为被动式的CID更新。

QUIC中采用的是主动式的CID更新模式，具体来说：QUIC要求一端需要保证对端有充足多的未使用过的CID，即在进行首次CID协商后，CID提供方需要主动通过QUIC的NEW_CONNECTION_ID帧来生产新的CID供对端使用。QUIC的这种主动式的CID更新机制要比被动式的CID更新要更robust一些。

问题：

- **生产方什么时候生产CID？**协议并没有严格规定，倒是提了一点：如果对端retire了CID，那么本端应该要生产并提供新的CID
- **生产多少CID? **这点受active_connection_id_limit 这一transport parameter 控制，双端在连接建立的时候宣告各自的active_connection_id_limit给对方，CID生产方生产的CID数量不能超过对端的active_connection_id_limit限制。

### Connection ID 退休

QUIC提供了Connection ID退休机制，具体是CID消费端可以通过RETIRE_CONNECTION_ID帧来让对端之前生产的CID失效。为了方便retire CID，QUIC将所有CID关联一个seq number，这样RETIRE_CONNECTION_ID中只需要带上一个seq 字段就能这个之前的所有CID退休。此外，生产方在NEW_CONNECTION_ID帧里也能带上一个seq，做到主动退休一些CID。

问题：

- **消费方什么时候退休CID ?**  总的来说，当消费方主动发起连接迁移后（比如无线切wifi后用新的local address发送packet）需要启用新的CID，那么此时也需要retire之前的CID，告诉对方之前的CID不会再用到了。
- **生产方什么时候退休CID ?** 消费方的连接迁移是被动发起的，比如因为NAT rebinding产生的连接迁移，此时消费方没有意识到自己的地址发生变化（local address没变），但是生产方收到包后感知到来自新的remote address，那么生产方需要自己retire这个CID并告诉对方。



### Connection ID认证

最后写点关于双端怎么去认证ConnectionID。首先为什么需要对Connection ID进行认证？由于QUIC中是在QUIC packet header中明文协商CID，因此需要注意额外对CID进行认证，否则协商的CID可能是被篡改过的。

认证的思想很简单，就是将CID作为放到传输层参数中交给TLS stack去认证。具体来说是每端将自己的SCID放到initial_source_connection_id传输层参数中，交给TLS stack，再通过initial packet发送给对端。这样如果MITM对任何一端的SCID进行任何篡改都会在收到并处理TLS的CertificateVerify消息或者FINISHED消息时被发现。对于client的initial packet中的DCID，QUIC也会进行认证，做法是让server 在inital packet的传输层参数中带上original_destination_connection_id。

## 2. Connection  Setup

QUIC的连接建立过程主要就是QUIC利用TLS stack协商密钥过程，这些内容在QUIC Cryptographic hanshake 中已经详细介绍过了，因此重复内容就不再赘述，本章主要写点RFC9001 中没有提到而是在RFC9000中定义的功能和行为。主要有建立连接时的可能涉及到的地址验证和版本协商过程。

### Address Validation When Setup

在DTLS1.3中Address validation 通过复用TLS1.3的HelloRetryRequest消息来做到。QUIC中并没有这么做，HelloRetryRequest对于QUIC来说和其他TLS消息一样没有区别，QUIC设计了一套自己的Address validation机制。本节将详细叙述下这个机制。

和DTLS一样，Address validation主要是为了防范反射攻击，QUIC规定在连接建立和连接迁移时都需要对一个未经验证过的地址进行Address validation。

QUIC 规定在未对对端地址进行验证之前，一端不能发送超过3倍于它接收到的数据，这个限制称之为anti-amplification limit。为了不让server端阻塞在这个限制上，QUIC要求Client端的Initial packet的UDP包大小至少要是1200个字节，不够的话需要padding，这样server能够在验证地址前发送更多的数据。注意这里可能会造成一个死锁的情况：如果server的Initial/ handshake包丢失了，并且达到了anti-amplification limit，此时client端如果收到了来自server的ACK，但是没有收到足够的后续的initial/handshake包，client将不会再发送任何数据，server端由于达到了limit也不会再发送任何数据，为了打破这个死锁，QUIC要求client要在probe  timeout时重传packet。

由于连接建立过程本身就隐式包含了地址验证，因此QUIC不像DTLS那样必须要在密码学握手前发起一个HelloRetryRequest。比如实现可以认为一旦一端完成了密码学握手过程，那么就完成了对端的地址验证过程。或者收到了对端使用本端选择的CID的包，也可以认为对端完成了地址验证。 

当然，server也可以选择在进行密码学握手前进行显式的地址验证。QUIC提供了类似于DTLS采用的stateless retry机制。


### Retry 机制

Retry的内容比较多，因为单独拎出来成为一节。总的来说QUIC的Retry机制和DTLS采用的stateless retry类似，不同在于QUIC中token（DTLS中叫做cookie）可以是server 在Retry packet告诉client，也可以是在之前的QUIC connection通过NEW_TOKEN帧前提下发给client的。

- Server的Retry packet只有header，并且没有header protection，但是有个retry integrity tag 字段用来让client校验retry packet 的integrity。其生成过程在[RFC9001]()的5.8节中定义，简单来说就是用一个固定的key和nouce去AEAD加密空白串，AD采用的从Retry packet header构造的Retry Pseudo-Packet，其实就是利用AEAD生成一个authentication tag。

```JavaScript
Retry Packet {   
 Header Form (1) = 1,   
 Fixed Bit (1) = 1,   
 Long Packet Type (2) = 3,   
 Unused (4),   
 Version (32),   
 Destination Connection ID Length (8),   
 Destination Connection ID (0..160),   
 Source Connection ID Length (8),   
 Source Connection ID (0..160),   
 Retry Token (..),   
 Retry Integrity Tag (128),
}
```
- QUIC 并没有规定怎么构造这个token，具体可以参考DTLS的做法。
- 注意Retry packet是唯一没有PN的packet，因此无法ACK。
- Client在收到Retry packet后，需要重新发送带上这个token 的initial packet / 0-rtt  packet，token在initial packet的 header中。需要注意的是Client必须不能去重置packet的PN space。

```JavaScript
Initial Packet {   
  Header Form (1) = 1,   
  Fixed Bit (1) = 1,   
  Long Packet Type (2) = 0,   
  Reserved Bits (2),   
  Packet Number Length (2),   
  Version (32),   
  Destination Connection ID Length (8),   
  Destination Connection ID (0..160),   
  Source Connection ID Length (8),   
  Source Connection ID (0..160),   
  Token Length (i), //如果length为0，表示没有token   
  Token (..),   
  Length (i),   
  Packet Number (8..32),   
  Packet Payload (8..), 
 }
```
- Server在收到Client的带有token的initial packet后，要么接受（token验证通过），要么拒绝，不得再发起一次Retry packet。


### Address Validation for Future Connections

在介绍这个之前，我个人其实有些疑问的: future connection一般都会采用psk handshake，既然都采用psk handshake了，说明地址已经验证过了，为啥还需要再来一次呢？

QUIC server可以在一个已经建立的连接中通过NEW_TOKEN帧下发token给client，client在后续的握手的initial packet中带上token给server进行地址校验。

和Retry packet中的token不同的是，NEW_TOKEN产生的token要有比较长的有效期，能够在client在后续的握手中带上时还生效。Client应该注意不要去重用token，否则可能会被path上的第三方通过token关联到身份。

Server端在生成token的时候要区分Retry packet的token和NEW_TOKEN frame。对于client在initial packet带过来的是NEW_TOKEN frame生成的token，如果校验不通过， server应当只将client视为未验证地址的client，然后继续处理。如果是retry packet验证不通过则直接拒绝。


### Version Negotiation

QUIC由于一直要进行协议迭代升级，因此会存在多个版本共存的情况，所以QUIC提供了版本协商机制。QUIC的版本协商不同于TLS的版本协商机制，可以说不那么直观，甚至有点拧巴。

QUICV1这本版本里要求，如果客户端支持多个版本的QUIC，那么选择最大的 minimum packet sizes作为第一个initial packet的大小。server通过这个size来决定是否进行版本协商，如果server不接受这个版本，会通过一个额外的Version Negotiation 包将自己支持的版本告诉Client，当然这也将会产生一次额外的RTT。（为什么这么做？initial packet里不是有version字段么）

Version Negotiation包目前没有加密保护，后续的版本可能会将其纳入密码学握手过程中。

整体看起来当前QUIC V1中并没有完善规定版本协商的什么时候进行、以及怎么进行。


## 3. Connection Migration

前面提到过了，连接迁移分为主动连接迁移和被动迁移。二者的区别仅限于发起端知不知道。


### 连接迁移过程

- 首先连接建立过程中不允许连接迁移，另外如果对端设置了disable_active_migration transport parameter，本端也不能发起主动连接迁移。
- 发起主动连接迁移不需要事先和对端进行协商交互，只需要在新的local addr上向对端发送non-probing packet即可（包含普通的数据、控制帧的packet）。迁移后CC、RTT、ECN等状态需要重置。
- 对端收到感知到连接迁移后，对non-probing packet的响应数据必须要发送到新的地址，并且必须要发起针对这个新地址的Path validation（如果这个地址未经validate的话）。在地址未经验证前，接收端能够发送的数据量将会受anti-amplification limit限制。一旦确认了迁移后的新地址，CC、RTT等状态也需要重置（如果对端只是换了端口，那么很可能是只是发生了NAT rebing，这时本端可以选择保留这些状态）

### 可能会受到的几种攻击

- Peer address spoofing：对端用假的地址（反射攻击受害者的地址）进行连接迁移， Path validation可以防范这类攻击
- On-Path Address Spoofing： 说的是双端路径上的攻击者（MITM），通过copy真实的UDP 包，然后篡改source address进行伪造连接迁移。为了保护连接不被这样的攻击所中断，在Path validation失败后，接收端需要将连接状态revert成上一个验证过的地址上的状态。
- Off-Path Packet Forwarding： 说的是双端路径外但是能够观测到packet的攻击者，通过不断copy并转发真实的包，来诱发将连接迁移到攻击者自己身上，使自己成为on-path 攻击者，从而能够干扰并丢弃所有后续的通信。这种攻击对于数据包较少的连接比较容易成功，因为如果原来的path上再有新的packet发送的话，连接还能再迁移回来，因此，这种攻击可以通过触发真实的数据包交互来进行缓解。因此QUIC规定，接收端感知到迁移后，还需要对原地址进行一次Path validation，如果一端在active path上收到了PATH_CHALLENGE，需要回应一个non-probing packet。这样便能够从attack手里夺回连接。当然这个措施也不是完美的，实现上还可以采用一些启发式的策略（比如：如果是NAT rebind的话，那么旧的path是后面不太可能还会发送数据；IPV6的path不太可能产生NAT rebind；如果connection migration的同时 packet的CID也变了的话，那基本上就是正常的连接迁移而非攻击）


### Path Validation

**what:** 这里的Path指的就是传统UDP的四元组，一个Connection可能存在多个Path，

**when:** Path validation一般发生在一端感知到对端发生连接迁移后，用于确保对端新的地址是真实有效的。也能发生于连接的任何时候，比如当对端已经休眠一端时间后，本端想知道对端地址是否还有效。再比如本端想发送主动的连接迁移前，可能会先进行一次path 探测。

**how:**

- 一端通过PATH_CHALLENGE frame来发起path validation，另一端需要以PATH_RESPONSE帧echo 收到的challenge。成功验证PATH_RESPONSE 后才算成功path validation，收到PATH_CHALLENGE的ACK不算完成path validation，因为ACK帧信息墒太小，即便是加密的也可能被暴力伪造。
- PATH_CHALLENGE和PATH_RESPONSE都需要填充到至少1200个字节，用来确保MTU是足够的，除非当前连接被anti-amplification limit限制了。如果被anti-amplification limit限制了，在完成当前的path validation后还需要再进行一次path validation用来验证Path MTU。
- 如果PATH_RESPONSE超时没有收到，那么此次path validatioon被认为失败。这个超时时间要比当前的PTO设置的大，推荐3倍于当前PTO。失败不代表就要中断连接，而是需要继续在其他path上发送数据，如果没有可用的path，实现可以选择等待path可用或者中断连接。


### Server Preferred Address

Preferred Address是server在握手阶段告诉client的传输层参数，用于让client在和当前地址完成握手后，将连接迁移到新的server端地址上（这个功能用在什么场景下？）。

一旦client和server的当前地址完成握手，client就需要发起到Preferred Address的path validation，当path validation成功完成后，client将在新的地址上发送后续所有数据。如果path validation失败，则继续使用原来的地址。



## 4. Connection Termination

QUIC连接可以用三种方式关闭：

- idle timeout : 空闲超时触发关闭，类比TCP  keepalive
- immediate close： 主动挥手关闭，类比TCP的四次挥手
- stateless reset：类比TCP RESET

### Idle Timeout

连接建立的时候双方可以设置max_idle_timeout这个连接层参数来告诉对端：如果在连接空闲时长超过max_idle_timeout，那么连接将会被本端主动关闭（发起一个immediate close）。max_idle_timeout可以为0，表示本端不会进行空闲超时关闭连接。



如果想要保持连接不被超时关闭，一端应该定期发送PING 帧来让连接保活。一方面是为了避免连接因为空闲超时被QUIC层关闭，另一方面，网络中间件比如 NAT对UDP包的处理会比TCP更激进，需要更短的包间隔来维持NAT状态，一般30s的超时能够应付大部分的中间件。



### Immediate Close

QUIC规定如果一端想要关闭连接，需要发送CONNECTION_CLOSE帧给对端，CONNECTION_CLOSE的发起方进入closing state，接收方收到后进入draining state。QUIC规定这些状态至少都要保留3个PTO。（相比于TCP，time_wait则需要保留一2MSL）


在**Closing state**下对任何packet都只回应CONECTION_CLOSE帧，所以endpoint只需要保留那些足以产生CONNECTION_CLOSE的状态就行，比如endpoint端生产的CIDs以及QUIC version，对端的QUIC packet过来后只需要从header里读取出CID、就能决定是否回应以CONNECTION_CLOSE。


当endpoint收到CONNECTION_CLOSE后进入**Draining state**，Draining state不允许发送任何 packet，静静等待连接关闭。


在握手阶段就要关闭连接的话需要考虑将CONNECTION_CLOSE帧封装在什么packet里，可能会涉及到发送多个不同加密等级的包含CONNECTION_CLOSE的packet，以确保对端能够正确收到。


最后说一点：Immediate Close一般是由应用层主动告诉QUIC 层（此外，QUIC层任一端如果感知到违背协议的行为时也会导致Immediate Close)，因此在发起Immediate Close之前应用层应当和对端协商好进行graceful shutdown的操作（比如关闭所有stream），Connection层不再和TCP那样有half-close这个中间态，并且需要经过4次挥手来保证双方都确认对端关闭，而是说立即关闭就立即关闭。


### Stateless Reset

和TCP类似，QUIC也有个Reset机制，作为处理从那些不认识的连接上过来的packet的最后手段。通常而言，如果一端因为crash或者断电重启后所有连接状态都将丢失，对端由于不知道还会继续发送数据包，此时本端只能用Reset机制来告诉对端连接已经不在了。


我们知道TCP一个非常易被干扰的地方就是被注入RESET包，因此在QUIC里为了避免这个问题，设计上只有对端能进行RESET。为了做到只让对端RESET，就必须在RESET包里携带一个只有通信双方才知道的reset token，这个token必须要足够难猜，QUIC规定长度需要是16 bytes。


reset token和CID是一一对应绑定的，CID的生产方在使用NEW_CONNECTION_ID生产CID的时候需要附带这个reset token给接收方，也会在握手时候通过stateless_reset_token传输层参数携带握手SCID对应的token。这个reset token也是由用CID进行加密产生，加密的使用的key将会是一个static key，为了让reset token生产方能够在重启后不需要依赖别的任何状态就能通过CID计算对应的token，然后便能对对端过来的packet进行安全的stateless reset。


为了让Stateless Reset在第三方眼里看起来和普通的short header包没有区别，reset包里除了带上根据收到的packet里的DCID计算得到的token外，还要带有一个随机生成的DCID。这也导致了两个问题：

- reset 包可能不能成功实际到达对端，如果对端严重依赖DCID来做路由的话
- 随机生成的DCID也有可能会被第三方识别到这是一个可能的reset包。

除此之外，reset packet需要占据整个UDP包，token位于包的最后。这也接收方就可以根据最后的16字节识别到reset。endpoint 可以选择对所有inbound 的UDP包都做reset 检测，但是需要注意防止时间侧道攻击。一旦正确检测到reset，发送端需要进入draining state，停止发送数据、等待连接结束。


最后，stateless reset可能会导致双方进入循环reset交换，因此QUIC规定每个stateless reset 包需要比触发它的packet要小（这样经过几次交换后的packet大小将不会触发stateless reset），除非有其他的防止looping的机制。





















