---
layout: post
title: "Using TLS to secure QUIC"
description: ""
category: 
tags: []
---
## Overview
这篇blog是关于QUIC协议如何使用TLS1.3来建立安全的QUIC连接，以及QUIC层如何对packet进行加密认证保护的内容，主要内容来自于[rfc9001](https://datatracker.ietf.org/doc/html/rfc9001)，结合自己的理解做了一些内容上的调整。原RFC文档个人认为写的非常全面易读了（至少比QUIC本身的RFC9000要易读多了。。），有兴趣的最还是要去读一读原文。

## 1 QUIC和tls1.3的关系

```JavaScript
+--------------+--------------+ +-------------+
|     TLS      |     TLS      | |    QUIC     |
|  Handshake   |    Alerts    | | Applications|
|              |              | |  (h3, etc.) |
+--------------+--------------+-+-------------+
|                                             |
|                QUIC Transport               |
|   (streams, reliability, congestion, etc.)  |
|                                             |
+---------------------------------------------+
|                                             |
|            QUIC Packet Protection           |
|                                             |
+---------------------------------------------+
```


和tls1.3 over tcp不一样，quic和tls1.3并非严格的上下层关系，而是quic连接的建立过程会复用tls1.3的handshake协议来进行密钥协商，tls1.3依赖quic transport提供的传输层可靠性保证。因此quic 和tls1.3的关系是同一个layer中的两个组件。

<!--more-->

```JavaScript
+------------+                               +------------+
|            |<---- Handshake Messages ----->|            |
|            |<- Validate 0-RTT Parameters ->|            |
|            |<--------- 0-RTT Keys ---------|            |
|    QUIC    |<------- Handshake Keys -------|    TLS     |
|            |<--------- 1-RTT Keys ---------|            |
|            |<------- Handshake Done -------|            |
+------------+                               +------------+
 |         ^
 | Protect | Protected
 v         | Packet
+------------+
|   QUIC     |
|  Packet    |
| Protection |
+------------+
```


quic中没有使用tls的record协议作为handshake、alerts、application数据的载体，因为quic packet本身就是载体。



## 2. TLS in QUIC

### tls1.3简化握手过程

```JavaScript
    Client                                             Server

    ClientHello
   (0-RTT Application Data)  -------->
                                                  ServerHello
                                         {EncryptedExtensions}
                                                    {Finished}
                             <--------      [Application Data]
   {Finished}                -------->

   [Application Data]        <------->      [Application Data]

    () Indicates messages protected by Early Data (0-RTT) Keys
    {} Indicates messages protected using Handshake Keys
    [] Indicates messages protected using Application Data
       (1-RTT) Keys 
```


### quic 简化握手过程

```纯文本
client                                                  Server

Initial[0]: CRYPTO[CH] ->

                                 Initial[0]: CRYPTO[SH] ACK[0]
                       Handshake[0]: CRYPTO[EE, CERT, CV, FIN]
                                 <- 1-RTT[0]: STREAM[1, "..."]

Initial[1]: ACK[0]
Handshake[0]: CRYPTO[FIN], ACK[0]
1-RTT[0]: STREAM[0, "..."], ACK[0] ->

                                          Handshake[1]: ACK[0]
         <- 1-RTT[1]: HANDSHAKE_DONE, STREAM[3, "..."], ACK[0]
```


#### 概念1： 加密级别 （encryption level）

我们知道QUIC中所有数据都是加密认证的，但是在连接的不同阶段加密认证的等级是不一样，这也很容易理解，比如对于一个全新的quic连接建立而言，第一个数据包不可能做到高等级的加密，因为这时候双方此前没有经过任何的密钥交互，quic 称这个数据包加密级别为Initial，采用的密钥是协议版本相关的静态密钥。当server将收到clientHello消息（client通过Initial包里的CRYPTO帧发给）后，按照tls1.3协议此时server就可以计算出Handshake secret了，那么后面数据包的加密等级就可以升级为Handshake级别了。等到双方完成握手，加密等级再次升级为1-RTT级别。



不同的加密级别对应的密钥不同，而quic包有可能会乱序到达，因此对端收到包后需要知道包使用的密钥才能正常解密，因此quic将不同的加密级别的包进行区分，对应不用的包类型(packet type)。

- Initial: 采用initial secrets加密 
- 0-RTT protected: 采用0-RTT  secrets加密，其实就是tls里的PSK密钥
- Handshake: 采用tls handshake secrets加密
- Retry: Retry secrets (这是什么？)
- Version Negotiation: 版本协商包比较特殊，不加密
- Short Header:  short header包就是完成连接握手后的包，也称为1-RTT包，采用的是正常的协商出来的密钥加密，quic里称为1-RTT secrets



#### 概念2： PN 空间 （Packet number space）

quic中每个包都有一个唯一的packet number，用来标识这个quic packet，而且quic的packet number严格单调递增，意味着即便是包丢失导致重传，quic也都是采用一个递增的PN，承载所丢失的stream frame数据，这样做的好处是避免tcp里在预估rtt时的重传歧义问题。

所谓的PN space就是PN的取值空间，quic协议将采用不同加密级别的包划分到几个独立的PN space中，这样做的好处主要是为了将握手包和数据包的PN space区分开，期望简化loss detection和congestion controller的实现([https://www.youtube.com/watch?v=mDc2kHPtavE&ab_channel=DanielStenberg](https://www.youtube.com/watch?v=mDc2kHPtavE&ab_channel=DanielStenberg) @11:06)。



#### 概念3: CRYPTO frame

TLS协议假设传输层是有序的，因此QUIC中 通过CRYPTO frame专门携带tls handshake消息。CRYPTO帧可以看作是一种特殊的不带stream id的STREAM帧，和STREAM帧一样和有offset、length、data等属性，因此可以做到有序交付给TLS。所有CRYPTO消息发送在这个特殊的stream上以保证握手消息的有序性。CRYPTO帧不受flow controller控制（但因该会受congestion controller控制）。

由于0-RTT类型的包是用来传输应用数据，所以虽然0-RTT包在握手过程中传输，但是QUIC不会用CRYPTO帧来封装0-RTT数据，而是采用常规的STREAM帧。

由于在握手完成后TLS可能还会有些post-handshake消息，比如NewSessionTicket，所有1-RTT包里也可能会有CRYPTO帧。因此CRYPTO帧会发送在Initial、Handshake、1-RTT包中。

### QUIC和TLS的交互过程

QUIC规定了一些和TLS交互行为。



**握手过程中**主要有：

- QUIC在开始TLS握手前需要向TLS提供自己的传输层参数（会在TLS ClientHello消息里作为tls extension发送）
- QUIC通过向TLS请求 握手数据来启动握手过程。TLS stack维护当前的sending encryption level以及receiving encryption level，当QUIC向TLS请求握手消息数据时，TLS会将sending encryption level附带上，QUIC根据encryption level来选择packet type，以及决定用什么key来对packet进行protect，然后发送给对端。
- QUIC收到对端握手数据后，对数据进行deprotect处理后向TLS提供 收到的握手数据。具体来说，当一端收到包含CRYPTO frame的QUIC packet时，将会做如下处理：
	1.  如果这个包采用的是TLS当前的receiving encryption level，说明这个包也正是TLS期望收到的握手消息，那么将这个CRYPTO frame按照frame的offset/length属性进行排序处理就行，再有序交付给TLS。
	2. 如果这个包比TLS当前的receiving encryption level旧，那么这个包肯定是之前数据的重传包，丢弃即可
	3. 如果这个包要比TLS当前的receiving encryption level新，那么说明这个包是乱序先到达的包，先存起来供TLS后续处理。一旦TLS告知QUIC receiving encryption level前进了，QUIC再将这个包的数据有序提供给TLS
- 每次QUIC提供数据给TLS后，都需要再向TLS请求新的握手数据，直至完成握手。
- QUIC和TLS间交互方式除了QUIC向TLS请求数据或者向TLS提供数据外，当一个新的encryption level可用时，TLS也会主动告诉QUIC这个encryption level的信息，QUIC依赖这些信息加解密packet。encryption level信息包括：这个encryption level的的secret, 协商的AEAD function和KDF function



一旦一端收到并验证了对端的tls Finished消息、并且发送了本端的tls Finished消息，那么对于该端来说握手就已经完成。QUIC还规定了在TLS**握手完成时**的交互行为：

- server端一旦握手完成，意味着server收到了client的Finished消息，此时双端的TLS握手都已经完成，server端确认了握手完成。QUIC规定server端必须尽快发出HANDSHAKE_DONE帧给Client。但是协议并没有规定TLS怎么告诉QUIC握手已经完成，通过encryption level change? Client收到server的HANDSHAKE_DONE帧后也被认为确认了握手完成。为什么要有这个HANDSHAKE_DONE帧？



**握手完成后**，TLS进行被动状态， 意味着TLS可以接受CRYPTO帧的数据，但是一般不会再有新的数据需要通过QUIC发送，除非应用层或者QUIC主动请求要发送一些数据，比如NewSessionTicket消息。



总结下：

```纯文本
Client                                                    Server
======                                                    ======

Get Handshake
                     Initial ------------->
Install tx 0-RTT keys
                     0-RTT - - - - - - - ->

                                              Handshake Received
                                                   Get Handshake
                     <------------- Initial
                                           Install rx 0-RTT keys
                                          Install Handshake keys
                                                   Get Handshake
                     <----------- Handshake
                                           Install tx 1-RTT keys
                     <- - - - - - - - 1-RTT

Handshake Received (Initial)
Install Handshake keys
Handshake Received (Handshake)
Get Handshake
                     Handshake ----------->
Handshake Complete
Install 1-RTT keys
                     1-RTT - - - - - - - ->

                                              Handshake Received
                                              Handshake Complete
                                             Handshake Confirmed
                                           Install rx 1-RTT keys
                     <--------------- 1-RTT
                           (HANDSHAKE_DONE)
Handshake Confirmed
```




### Session Resumption in QUIC

Session Resumption 在QUIC完全交给TLS stack处理，QUIC server端由TLS stack产生NewSessionTicket消息（一般session ticket里编码了session状态信息），然后通过CRYPTO帧发送给QUIC client端，client端收到后deprotect再交给TLS stack保存（通常来说，TLS stack会保存当前sesstion ticket以及对应的tls session状态信息）。整个过程client端的QUIC stack不需要保存任何状态信息。

### 0-RTT in QUIC

QUIC 中启用0-RTT基本和TLS1.3中0-RTT一样，数据通过PSK 加密（PSK通常由NewSessionTicket消息生产，因此0-RTT依赖于Session Resumption）。首先client在TLS的ClientHello消息中带上early_data 扩展，表示client接下来希望发送0-RTT数据包。紧接着client开始发送0-RTT包。然后server在ServerHello之后的EncryptedExtensions消息中带上加密的early_data扩展，告诉client自己接受0-RTT数据（如果没有带上，表示拒绝0-RTT数据）。



QUIC 0-RTT区别于TLS1.3 0-RTT的地方主要有：

- 在TLS1.3中server用newSessionTicket里的early_data扩展中的max_early_data_size字段来表示能接受的最大early_data大小。QUIC中不再称之为为early_data，而是用0-rtt包来传输0-rtt数据，因此QUIC server用initial_max_data这一传输层参数来表达能接受的最大的0-rtt 数据大小
- QUIC由于0-RTT有单独的包类型，所以不再需要通过发送TLS的EndOfEarlyData消息来告诉对端0-RTT数据结束。
- 最重要的区别是：TLS1.3中接受0-RTT数据与否完全取决于TLS stack，QUIC中不仅取决于TLS组件，也要取决于QUIC stack。意味着现在即便tls stack没有拒绝，quic stack也有权拒绝0-rtt data。这么做的原因是在一个新连接上直接发送0-RTT包的前提是这个新连接上ClientHello里带的传输层参数需要和之前旧连接协商出的传输层参数得一致才行。同时也意味着如果启用QUIC 0-RTT，QUIC stack也需要保存一些传输层相关的session状态。对于QUIC  server端来说，这意味这session ticket里需要额外编码传输层状态，对于QUIC client来说，在收到NewSessionTicket消息的CRYPTO帧时，除了交给TLS stack保存TLS session状态外，QUIC stack也需要存储session ticket以及对应的session传输层状态。



### 密钥卸载时机

1. **Initial keys卸载**： Initial keys是可以被攻击者构造的，因此需要尽快卸载。QUIC规定client端在第一次发送Handshake包后就需要卸载Initial keys（意味着client收到了server 发的Initail& Handshake包, 因此client端不再需要Initial keys了）。server端在第一次接收并成功处理Handshake包后就需要卸载。
2. **Handshake keys卸载**:  QUIC规定Handshake keys在双方都确认handshake完成后（server 发送了Handshake_done，client收到了server 的Handshake_done）卸载。
3. **0-RTT keys卸载: ** 对于Client来说，当收到了server的Handshake消息并成功处理后，此时client会安装1-rtt keys，后续将只会发送1-rtt 包（即便之前发送的0-rtt data丢失了，由于0-rtt和1-rtt 包共享同一个PN space，后续重传的话也是采用1-rtt包）。对于Server端来说，建议在收到1-rtt包之后的一小段时间后卸载掉0-rtt keys（3倍PTO时间），为什么不建议立即卸载呢，因为有可能0-rtt 包乱序到达，比1-rtt还要晚，如果卸载了那么由于解密不了后到达的0-rtt包，client只能重传了。 



## 3. QUIC Packet Protection

前面提到，quic不再使用tls的record协议，而是将tls record层做的事情挪到了quic packet上，利用tls handshake协商出的各种密钥、AEAD、KDF函数来对packet进行密钥学保护。



QUIC中几乎所有的packet都做了密码学保护，不同packet类型的保护级别不一样（前面也提到了）。具体来说有：

- Version Negotication packet 不进行密码学保护
- Retry packet使用静态密钥 + AEAD_AES_128_GCM进行保护，为了防止不认识QUIC的中间件不小心的修改
- Initial packets使用从client的initial packet中的Destination CID派生的密钥 +  AEAD_AES_128_GCM进行保护。保护的力度和Retry packet差不多（不能保证confidentiality  和 integrity）
- 剩下其他的packet 都采用tls handshake协商&调度出的密钥 + 加密算法套件进行强密码学保护。



这里回顾下TLS key schedule过程如下：

```JavaScript
             0
             |
             v
   PSK ->  HKDF-Extract = Early Secret
             |
             +-----> Derive-Secret(., "ext binder" | "res binder", "")
             |                     = binder_key
             |
             +-----> Derive-Secret(., "c e traffic", ClientHello)
             |                     = client_early_traffic_secret
             |
             +-----> Derive-Secret(., "e exp master", ClientHello)
             |                     = early_exporter_master_secret
             v
       Derive-Secret(., "derived", "")
             |
             v
   (EC)DHE -> HKDF-Extract = Handshake Secret
             |
             +-----> Derive-Secret(., "c hs traffic",
             |                     ClientHello...ServerHello)
             |                     = client_handshake_traffic_secret
             |
             +-----> Derive-Secret(., "s hs traffic",
             |                     ClientHello...ServerHello)
             |                     = server_handshake_traffic_secret
             v
       Derive-Secret(., "derived", "")
             |
             v
   0 -> HKDF-Extract = Main Secret
             |
             +-----> Derive-Secret(., "c ap traffic",
             |                     ClientHello...server Finished)
             |                     = client_application_traffic_secret_0
             |
             +-----> Derive-Secret(., "s ap traffic",
             |                     ClientHello...server Finished)
             |                     = server_application_traffic_secret_0
             |
             +-----> Derive-Secret(., "exp master",
             |                     ClientHello...server Finished)
             |                     = exporter_secret
             |
             +-----> Derive-Secret(., "res master",
                                   ClientHello...client Finished)
                                   = resumption_secret
```


注意到上面调度过程TLS为Handshake和1-rtt 数据都派生client和server两个secret。有了secret后，在原来TLS协议中还需基于这个secret派生出key和iv，tls中通过下面方法派生得到：

```JavaScript
   [sender]_write_key = HKDF-Expand-Label(Secret, "key", "", key_length)
   [sender]_write_iv  = HKDF-Expand-Label(Secret, "iv", "", iv_length)
```


**QUIC中将由secret派生出key和iv这部分工作挪到了QUIC层**，另外还会新增派生了header protection key（后面会介绍）。具体通过下面得到：

```JavaScript
   [sender]_write_key = HKDF-Expand-Label(Secret, "quic key", "", key_length)
   [sender]_write_iv  = HKDF-Expand-Label(Secret, "quic iv", "", iv_length)
   [sender]_header_protection_key = HKDF-Expand-Label(Secret, "quic hp", "", key_length)
```


### Initial Secrets 生成

initial secrets由于是QUIC中引入的， 和0-rtt、handshake、1-rtt secrets不一样，后者是由TLS stack生产。Initial secrets则由QUIC通过client initial packet中的Destionation CID采用TLS的提供的HKDF_Extract和HKDF-Expand-Label函数派生&扩展得到，具体如下所示：

```JavaScript
      initial_salt(0x38762cf7f55934b34d179ae6a4c80cadccbb7f0a)
             |
             v
  D_CID -> HKDF-Extract = Initial Secret
             |
             |
             +-----> HKDF-Expand-Label(., "client in", "", Hash.length)
             |                     = client_initial_secret
             |
             +-----> HKDF-Expand-Label(., "server in", "", Hash.length)
                                   = server_initial_secret
```


这里有两点需要注意：

- Destination CID通常是由client 随机选择，如果server通过Retry包要求client去重发Initial packet，此时Destination CID将由Server生成。
- initial_salt每个quic版本将定义不同的值，以保证每个QUIC版本的Initial secret不同。通过这样做来对抗那些可能认识并且修改某一个版本的QUIC Initial secret的middlebox。



### AEAD使用

QUIC支持TLS1.3中除了TLS_AES_128_CCM_8_SHA256之外的其他AEAD cipher suites，这些AEAD 函数的都将输出一个16字节大小的authentication tag，因此最终加密得到的密文要比输入的明文大16个字节。 

我们知道AEAD函数加密的定义形如：

```
AEADEncrypted = AEAD-Encrypt(write_key, nonce, additional_data, plaintext)

plaintext of encrypted_record =        
                AEAD-Decrypt(peer_write_key, nonce, additional_data, AEADEncrypted)
```


在TLS中, nonce通过tls的`seq_num XOR (client_write_iv / server_write_iv)` 得到，QUIC中没有seq_num，因此很自然的采用的是packet number。additional_data则采用的当前QUIC packet的 header（从header的第一个字节开始到packet number最后一个字节结束）

## 4. QUIC Header protection

### Header protection保护范围

**Long header**的定义如下：

```纯文本
Long Header Packet {
  Header Form (1) = 1,
  Fixed Bit (1) = 1,
  Long Packet Type (2),
  Type-Specific Bits (4),
  Version (32),
  Destination Connection ID Length (8),
  Destination Connection ID (0..160),
  Source Connection ID Length (8),
  Source Connection ID (0..160),
  Type-Specific Payload (..),
}

其中不变部分可视化表示如下：
0                   1                   2                   3     
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1    
+-+-+-+-+-+-+-+-+    
|1|X X X X X X X|    
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Version (32)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
| DCID Len (8)  |    
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
|               Destination Connection ID (0..2040)           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
| SCID Len (8)  |    
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
|                 Source Connection ID (0..2040)              ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
|X X X X X X X X X X X X X X X X X X X X X X X X X X X X X X  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```


上面这些字段是QUIC 所有packet type的long header共有的，除了这些字段外，header里往往还有几个常见的字段：

- Reserved Bits: 属于Type-Specific Bits一部分(前两个bits)，但是所有packet type都有，目前保留为0
- Packet Number Length: 如果packet 有Packet number，那么Type-Specific Bits的后两个bits就是Packet Number 的length
- Length： 表示packet里接下来数据的长度
- Packet Number： 不解释了

对于Long header而言，header protection保护的是 Type-Specific Bits (包含Reserved Bits和Packet Number Length) + Packet Number



**Short Header**定义如下：

```JavaScript
1-RTT Packet {
  Header Form (1) = 0,
  Fixed Bit (1) = 1,
  Spin Bit (1),
  Reserved Bits (2),
  Key Phase (1),
  Packet Number Length (2),
  Destination Connection ID (0..160),
  Packet Number (8..32),
  Packet Payload (8..),
}

其中版本不变的部分可视化为：
0                   1                   2                   3     
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1    
+-+-+-+-+-+-+-+-+    
|0|X X X X X X X|    
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
|                 Destination Connection ID (*)               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+    
|X X X X X X X X X X X X X X X X X X X X X X X X X X X X X X  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```


对于Short header，保护的是第一个字节的后5个bits (Reserved Bits + Key Phase + Packet Number Length) + Packer Number 



#### Header Protection 工作过程

首先header protection工作在packet protection之后，对packet protection得到ciphertext的进行采样作为加密算法的输入，加密算法依赖于协商得到的AEAD算法，加密算法输出是一个5-bytes的Mask，header protection过程就是将这个Mask和header中被保护的内容进行XOR。伪代码如下：

```JavaScript
mask = header_protection(hp_key, sample)

pn_length = (packet[0] & 0x03) + 1
if (packet[0] & 0x80) == 0x80:
   # Long header: 4 bits masked
   packet[0] ^= mask[0] & 0x0f
else:
   # Short header: 5 bits masked
   packet[0] ^= mask[0] & 0x1f

# pn_offset is the start of the Packet Number field.
packet[pn_offset:pn_offset+pn_length] ^= mask[1:1+pn_length]
```


#### 几点细节

- 采样内容和长度的规定：从PN字段开始处向后偏移4个字节才开始，偏移这4个字节是为了避开PN的任何字节（最大的PN长度就是4个字节）。heaeder protection sample 的长度是16个字节。因此header protection对packet payload的cipher text的长度也有要求。比如：如果PN长度是1个字节，那么这个cipher text 至少是3 + 16字节，这就要求payload的plain text至少得有3个字节（AEAD加密会扩充16个字节的authentication tag），不够的话就需要padding。
- 前面说到header protection的加密算法依赖于tls协商出来的AEAD算法，最终只有两类：AES-ECB和ChaCha20。在tls还没协商出cipher suite之前，默认采用的是AES-128-ECB 

总结一下整个packet protection过程：
![quic packted protection](/public/fig/quic_packet_protection.png)

## 5 QUIC Packet Deprotection

基于TCP的tls协议中，由于TCP层做了有序传输的保证，对端tls层在收到消息解密的时候，只需要采用当前secret就行了。到了QUIC这里，这个做法就不再work了。比如由于乱序导致Server端提前收到了下一个encryption level的包，此时前面提到过server端可以先buffer起来，等到tls层告知quic 提升encryption level后再deproect这个包。

```纯文本
Client                                                    Server
======                                                    ======

Get Handshake
                     Initial ------------->
Install tx 0-RTT keys
                     0-RTT - - - - - - - ->

                                              Handshake Received
                                                   Get Handshake
                     <------------- Initial
                                           Install rx 0-RTT keys
                                          Install Handshake keys
                                                   Get Handshake
                     <----------- Handshake
                                           Install tx 1-RTT keys
                     <- - - - - - - - 1-RTT

Handshake Received (Initial)
Install Handshake keys
Handshake Received (Handshake)
Get Handshake
                     Handshake ----------->
Handshake Complete
Install 1-RTT keys
                     1-RTT - - - - - - - ->

                                              Handshake Received
                                              Handshake Complete
                                             Handshake Confirmed
                                           Install rx 1-RTT keys
                     <--------------- 1-RTT
                           (HANDSHAKE_DONE)
Handshake Confirmed
```


这里有几个地方协议特别指出来：

- Server端先收到了0-rtt包，然后再收到Initial（clientHello）包：这种case下server可以选择将0-rtt包暂存，等到处理完Initial包之后，再去处理。
- client端先收到了server的1-rtt包，然后才收到 Initial/Handshake (ServerHello..Finished)包，此时client无法deproect server的1-rtt包，client也可以将其暂存。
- Server 端先收到了client的1-rtt包，然后才收到client的Handshake(Certificate..Finished)包，此时Server端理论上有1-rtt的secret，但是QUIC协议明确规定不能进行对1-rtt包的deprotect，必须要等到验证完client的Handshake后才可以处理client的1-rtt包。这也是为什么上图中server端在发送Handshake包后安装的是tx 1-rtt keys（发送端1-rtt密钥），只有当收到client的Handshake包并验证完成后才安装rx 1-rtt keys，此时才可以对收到的1-rtt进行deprotect并处理。这个规定有他的道理，毕竟在没有完成验证client handshake包前，是无法证明client的1-rtt包是安全的。但是这也给client的Handshake包和1-rtt包制造了QUIC packet间的HOL blocking。RFC也给了一个解决方案：client在发送每个1-rtt包时，在UDP报文里coalesce一个Handshake包的copy，直到client收到server端对Handshake包的ack。



## 6 Key update

TLS1.3中key update请求可由任一方发起，一旦发起表示后续sending方向的record采用新的secret派生出的keys加密。接受方接受到key update请求同步更新自己receiving方向的secret，然后根据请求里的KeyUpdateRequest标识决定是否响应自己sending方向的Key udpate。

显然TLS的这种key update机制依赖于Key update消息和使用新的secret的record消息的时序。QUIC里弃用了TLS的key update机制，而是采用short header中的Key Phase bit来标识是否要更新secret。Key Phase bit一开始置为0，后面每次更新secret时都flip下。



### 发起Key update

- 双方握手完成并确认后才能发起
- 发起方必须要收到过接收方的 使用当前secret的ACK packet （为了确保双方都有当前secret）
- 一旦一端发起Key update，不仅要更新sending方向的secret，也要更新接受方向的secret，这点QUIC和 TLS不同，强制双方同步更新两个方向的secret。
- 发起Key update并更新secret后将派生出新的keys & IV，后续发送packet将置位Key Phase bit，并且使用新的key去protect packet。但是仍然需要保留receiving方向上老的keys & IV一段时间，用来处理来自对端的in-fight和delayed packets。
- 注意一点: Key Phase bit也受header protection的保护，但是header protection key不会随着Key update而改变，在连接建立完成后header proection key将保持不变，这样使得接收方就能始终  deproect header，从而通过Key Phase bit知道什么时候做Key update。

### 接收并响应Key udpate

- 接收方第一次感知到packet的Key Phase bit发生改变时就需要同步更新receiving方向的secret，派生新的keys & IV, 然后用新的keys对当前packet进行deprotect。（注意这里的packet不一定是发送方使用新的key发送的第一个packet）。同理，更新receiving方向的keys后，并不是立即卸载receiving方向老的keys，也需要保留一段时间
- 如果成功deprotect这个packet，接收方进行Key update，更新发送方向的secret，派生新的keys & IV。后续发送所有packet都将使用新的key。当接收方使用新的key ack了第一个触发key update 的packet后，表示key update过程完成。

```纯文本
      Initiating Peer                    Responding Peer

   @M [0] QUIC Packets

   ... Update to @N
   @N [1] QUIC Packets
                         -------->
                                            Update to @N ...
                                         QUIC Packets [1] @N
                         <--------
                                         QUIC Packets [1] @N
                                       containing ACK
                         <--------
   ... Key Update Permitted

   @N [1] QUIC Packets
            containing ACK for @N packets
                         -------->
                                    Key Update Permitted ...
```


- 注意一点：当收到一个包的Key Phase bit和当前Key Phase不一样，不代表一定是对端发起了Key update，因为next Key Phase bit和previous Key Phase bit是一样的，有可能网络延迟导致收到的是previous key phase的包，因此需要区分这个包是previous key phase的还是next key phase的，只要next key phase时才去更新secret。怎么区分呢？可以Packet Number来判断：如果包的packet number低于当前key phase的所有packet number，那么表示收到的packet来自上一个key phase，相反如果大于当前key phase的所有packet number，那么表示收到的packet是对端进行了key update导致的下一个key phase的。
- 还要注意一点：在实现上需要防范时间侧道攻击，接收并响应key update过程从协议上来说很容易成为时间侧道攻击的口子，攻击者可以通过注入数据来观察接收端的处理时间来判断key phase，后面会介绍。

### 什么时候发起Key update

为了保证通信数据的confidentiality,  很多AEAD算法对同一个key 的使用次数是有限制的，这也是为什么TLS和QUIC里都有key update机制的原因。同样，通信数据的integrity保证也会要求很多AEAD算法限制同一个key的使用次数。在TLS协议里如果收到一个 record经过AEAD-decrypt失败（表示数据的integrity受损，可能被attacker篡改），那么TLS会立即中断连接。

QUIC里不一样，由于QUIC包的可能延迟达到，QUIC选择丢弃那些不能成功deprotect的包，这就给active attacker伪造packet创造了暴破攻击的条件，因为QUIC将AEAD的confidentiality和integrity的限制分开处理，confidentiality的限制应用在使用同一个key进行protect时，integrity限制应用在使用同一个key进行deprotect失败时。如果当前key使用达到confidentiality limit时，需要立即触发key update。如果key使用达到了integrity limit，那么QUIC需要中止使用connection。

- confidentiality limit大小
	- AEAD_AES_128_GCM：2^23
	- AEAD_AES_256_GCM:   2^23
	- AEAD_CHACHA20_POLY1305: 2^62
	- AEAD_AES_128_CCM: 2^21.5
- integrity limit 大小
	- AEAD_AES_128_GCM: 2^52
	- AEAD_AES_256_GCM: 2^52
	- AEAD_CHACHA20_POLY1305: 2^36
	- AEAD_AES_128_CCM: 2^21.5



## 7 MISC 

### QUIC对TLS握手过程的修改

- QUIC在握手时强制使用ALPN扩展进行应用层协议协商，如果协商失败，必须关闭连接
- QUIC新增quic_transport_parameters扩展来协商传输层参数，quic_transport_parameters扩展会在ClientHello和 EncryptedExtensions消息中强制携带，如果没有则关闭连接。
- QUIC删除了TLS握手的EndOfEarlyData消息，前面提到过QUIC不再依赖EndOfEarlyData消息来告诉对端early data结束。
- QUIC去掉了TLS的中间件兼容模式

### QUIC 协议安全性

首先所有TLS的安全性同样适用于QUIC，QUIC本身也会引入一些额外的安全性问题

- Packet Reflection Attack： 反射攻击说是的attacker通过使用被害者的地址来伪造源地址和端口，向server发起请求，QUIC协议中client以一个很小的ClientHello包就能引出server以相对大的handshake messages进行回应（发送数据给被害者），这种流量放大给attacker制造了反射攻击的便利，这个问题在QUIC上尤为明显，原因是QUIC基于 UDP，不像TLS over TCP需要经过TCP握手。为了缓解这个问题，QUIC规定：1） Initial packet（包含ClientHello消息）必须padding至协议规定的最小大小。2）对于一个未经验证的地址，server不允许发送超过3倍于收到数据大小的数据。这些在一定程度缓解了Packet Reflection Attack
- Header protection 安全性：在理论基础上，QUIC的header protection的算法构造采用的是[https://eprint.iacr.org/2019/624](https://eprint.iacr.org/2019/624)中提到的HN1构造，具备可证明的安全性。在实现上，QUIC将packet 经过AEAD加密后密文的采样作为nouce，对这个nouce进行再次加密得到一个mask，最终将要被保护的field和这个mask进行XOR。这种做法安全性的前提是nouce不一样，由于nouce是对packet AEAD密文的采样，这个nouce一样的概率很小，大概是1/2^64。
- Header protection的时间侧道攻击：Header 中的packet number和Key Phase bit 可能容易遭受时间侧道攻击。对于packet number，举个例子：如果QUIC实现在处理一个PN重复的packet时候不去进行packet deprotection，那么处理时间上可能就和其他包不一样，也就暴露了一个时间侧道的口子，所以QUIC要求实现需要对所有包统一进行header deprotection → packet numeber recovery → packet deprotection。对于Key phase，同样在处理key update包时如果选择即时生成新的keys，那么处理时间上和处理非key update包肯定也不一样，因此协议中要求实现需要避免Key update的时间侧道，一个建议的做法是在处理Key update包之前就预先生成好新的keys，处理包时只需要做一次keys  set切换。



