---
layout: post
title: "QUIC Loss Detection"
description: ""
category: 
tags: [QUIC]
---

总体上QUIC的重传机制吸取了大量TCP重传机制的经验，本章介绍的算法和机制基本上在TCP里都有对应的版本。但是QUIC和 TCP协议上的不同也体现在这些算法/机制上，其中最重要的一个不同点是，QUIC中数据传输和数据应用交付在协议上分开的，传输是发生在packet级别的，交付则是发生在stream级别，TCP中两者都是发生在byte stream级别。QUIC packet单调递增的seq number避免了重传歧义，这在TCP里是个比较麻烦的问题。



## 经典TCP重传机制

在正式介绍QUIC的重传机制前，我觉得最好先回顾下TCP对应的重传机制，这部分内容主要来源于《TCP/IP Illustrated》这本书，Richard Stevens大牛已经将TCP协议的big picture总结的明明白白，我们这样的后来人翻翻书就能避免钻进IETF的TCP“故纸堆”中迷失自我了。

TCP在认为发生丢包时通过重传来解决，因此我们说的重传机制其实最核心的就是TCP在什么情况下认为某个segment丢失了——即loss detection，至于怎么重传在TCP里面是个相对直观的问题，即原样重传那些认为丢失的segment即可，所以有时候会把重传算法和丢包检测算法这两个概念混用。另外TCP重传往往会牵扯到CC算法控制，这里先不做展开。

总的来说TCP经典的重传机制有两个：基于时间的重传（RTO recovery）和基于ACK结构的重传（fast recovery）

- 基于时间的重传机制：超时重传需要TCP连接上维护一个重传定时器，当TCP要发送一个新segment时会先取消老的定时器，设置上新的定时器（或者说更新定时器的超时时间），在收到新segment的ACK时取消定时器，如果在定时器到期前没有收到ACK，则触发超时重传，TCP将重传所有未被ACK的segments。这里需要注意的是TCP并不是为每发送的segment都维护一个重传定时器，而是一个连接上只会维护一个定时器，这么做的原因是对端也不是为每个segment都产生ACK，而是会delay cumulative ACK。超时重传对于TCP来说是非常影响性能的事件，第一它将触发CC算法最小化拥塞窗口、第二它将会增加超时时间的backoff。
- 基于ACK结构的重传机制：TCP规定接收方需要在收到一个乱序到达的segment时立即产生一个ACK（duplicate ACK）。发送方在收到dupthresh次duplicate ACK后将启动快重传，无需等定时器超时。dupthresh值一般是3次。发送方在首次进入快重传的时间点称为recovery point，此时发送方将记录下当前发出去最大的seq number，整个重传recovery过程将持续到收到的ACK匹配或者超过之前记录的最大seq。另外需要注意的是在recovery过程中收到ACK时sender也可能发送新的segment（拥塞窗口允许的情况下）。没有SACK option的情况下，一个RTT只能重传一个segment，如果双方都支持SACK，一个RTT内可以重传多个segment。

<!--more-->

### TCP RTT预估

TCP的超时重传机制中超时时间的设定依赖于连接的RTT，然而TCP协议面向的是字节流， cumulative ACK机制使得RTT采样这件事变得不那么容易。在没有开启Timestamp option的情况下，RTT采样只能针对整个窗口的所有segment来进行，因为发送端没法知道收到的ACK是在对端在收到窗口内哪个segment时产生的ACK。此外在有重传发生时收到ACK也不能作为RTT样本，因为有重传歧义问题。

有了Timestamp option，发送端在发送segment时将携带本端的一个时间戳，接收端在ACK是时候将segment的时间戳echo back给发送端，发送端收到ACK后用当前时间减去ACK中的时间戳就形成了一个RTT采样。Timestamp option的启用解决了cumulative ack和重传歧义问题导致的RTT预估不精确问题。当然接收端在ACK时候选择用哪个segment的时间戳是有讲究的，直接决定了对端RTT样本值。在启用Timestamp option情况下进行RTT采样的算法简单描述如下：

- 对于接收方，在产生ACK时用收到的**最后一个有序的segment的ts**来echo back。这么做对于数据包按序到达的情况，echo back的是最后一个收到的包的ts值，发送端在收到ACK后拿当前时间 - ts值就是一个没有delay的RTT采样。对于数据包乱序到达的情况，echo back的是并非最后一个收到包的ts(乱序的)，而是之前的包的ts，这个样计算得出的RTT是实际链路的RTT + 接受端delay的结果，这个delay也需要纳入RTT计算当中，否则体现到RTO上将会使得RTO偏小。
- 对于发送方，在收到ACK时只针对那些使得滑动窗口前进的ACK产生一个RTT采样

有了RTT样本之后，剩下的问题就是这么用这些样本来计算一个平滑的RTT，有多种方法，但基本思想都是对RTT样本进行指数移动平均来进行平滑。以standard method为例，采样下面的方法来计算RTO

```text
srtt <- (1-g) srtt + (g)M    //M是一个rtt样本，srtt是rtt的EWMA
rttvar <-  (1-h) rttvar + (h) (|M - srtt|)  //rttvar 是采用mean deviation的EWMA方法近似计算rtt的标准差
RTO = srtt + 4 * rttvar 

srtt初始值为M, rttvar初始值为 M/2, g为1/8, h为1/4。h比g值要大使得RTT在发生变化时RTO能更快变大。
```

Standard method逻辑上很简单直白，但是存在一些问题，比如：

- 如果RTT采样很频繁、RTT粒度细时，mean deviation移动平均将使得rttvar趋于最小化
- 如果RTT样本突然大幅减小时，rttvar会随之增大，体现到最终RTO很可能也是变大，这就有点反直觉了。



### TCP 启用SACK 时的重传机制

对于接收端，总的来说接收端可以在buffer里有乱序到达的segment任何时候产生SACK，接收端将最新收到的序号段作为第一个SACK block，并且也会放置一些之前的序号段提供一些SACK信息冗余。



对于发送端，在收到一个SACK后，可以将buffer里出现在SACK block里的segment标记上已被SACK，当发送端有机会重传数据时可以只重传那些未被SACK过的segment。



TCP里的SACK在协议中被认为是建议性质的，也就说说接收端在SACK了某个序号端后还能反悔，因此发送端不能在收到SACK信息后立即将buffer中对应的segment数据删掉。当然反悔机制在TCP实现中非常少见也不鼓励。



### TCP Spurious Timeouts and Retransmission

产生Spurious 重传原因可能有很多，比如乱序、ACK丢失、RTT突然增大超过了RTO等等。Spurious重传在无线网场景下发生很频繁，这个问题之所以重要是因为一旦发生将会对发送方性能产生很大影响。

TCP这么多年发展出了很多手段去解决这个问题，总的来说涉及两部分算法

- detection algorithm: 负责检测重传是不是Spurious的
- reposonse algorithm：Spurious重传发送后怎么去undo或者缓解由于超时重传导致的负面影响

下面列举几个常见的算法

#### DSACK option

DSACK option用于让发送端能够检测到Spurious重传的发生，需要双端都支持才能生效，简单来说就是接收端在产生SACK的时候将收到的重复的序号段通过SACK block告诉给发送端，即便这个序号段已经在当前cumulative ACK序号之前。 



#### Eifel Detection Algorithm

Eifel Detection算法工作依赖Timestamp option，原理非常简单，就是利用Timetamp option解决重传歧义的功能来做识别ACK是对重传segment的ACK，还是原始segment的ACK，如果是对原始的segment的ACK，那么说明重传是Spurious重传。

具体来说：当发送端决定要去重传某个segment时，记录下segment 中timestamp option的ts（TSV），当收到第一个能够包含segment的seq的ACK时检查下ACK中echo back的ts（TSER），如果TSER比TSV要小，说明ACK是对原始segment的ACK，此前的重传是Spurious重传。

Eifel Detection相比于DSACK能够在ACK丢失时候更具鲁棒性



#### Foward-RTO Recovery (F-RTO)

正常情况下发送端一旦超时，TCP将表现出go-back-N行为，即发送端从最小未被ACK的segment开始重传，每收到一个ACK，向前重传直至完成恢复。F-RTO修改了这个行为，使得发送端重传开始的收到第一个ACK后直接开始发送新的segment。然后检查后面的收到的第二个ACK：

- 如果任意一个ACK是duplicate ACK，说明中间还有空洞，重传的没问题
- 如果两个ACK都使得窗口前进了，说明没有丢包，前面的重传是Spurious的

显然F-RTO只能用于检测Spurious超时重传，相比于DSACK、Eifel Detection，检测时机比较晚

#### Eifel Response Algorithm

简单来说就是当检测到Spurious timeout采取一些措施主要包括

- 重置CC相关状态
- 重置RTO相关状态





# RACK-TLP丢包检测算法

 QUIC在丢包检测部分很大程度上借鉴了RACK-TLP算法的设计，因此有必要介绍这个相对新的TCP丢包检测/重传算法，本章节的内容来源于RFC8985。RACK-TLP是由google的工程师提出并在2021年被IETF标准化，目前已经被集成到FreeBSD、linux、windows的TCP实现中。RACK-TLP旨在改进并替代传统的重传恢复策略。

传统的重传恢复策略前面已经介绍了：首先发送方在收到3个duplicate ACK后会触发快重传，没有SACK情况下每个RTT恢复一个segment，这一策略称为DupAck counting。快重传的同时，发送方的CC window也会减半（之所以是减半而不是减为1是因为这种情况不那么严重，至少ACK clocking还在工作）。如果快重传没有被触发、或者触发也没有恢复所有的丢包，那么发送方将最后用RTO超时重传策略兜底，一旦触发RTO超时，发送端将从第一个未被ack的segment开始重传，并且CC window重置为1，进行CC慢启动过程。RTO设置前面也介绍了，是srtt + 4 * rtt_var，并且通常会以1秒为下界，因此RTO超时重传对性能有很大的影响。

传统的重传恢复策略在以下场景下表现不好：

- tail loss:  tail loss说的是一个flight的多个segments里尾部的segment发生丢失，这在一些request/response结构的数据交互场景下，tail loss往往只能通过RTO超时来重传恢复，因为tail segment loss无法触发快重传的条件
- restransmission loss: 在一些严重丢包的场景下重传segment也可能丢失，这种情况下快重传也是无能为力，只能通过RTO超时重传来恢复
- packet reordering： 当链路的乱序程度（degree of reordering）高于dupthresh时，快重传策略会导致Spurious重传。



### RACK-TLP设计概括

RACK-TLP旨在更高效应对上面提到的三个场景，设计上分为两部分：RACK 部分和TLP部分。

- RACK：这部分的设计原则是说，尽可能地利用最近的ACK （Recent ACK）来进行loss detect，从而做到在RTT时间尺度上恢复loss。
- TLP：这部分的设计原则是说，通过给对端温和的发送探测包，从而引出对端的ACK反馈，避免RTO超时。



#### RACK

RACK部分最核心的问题是怎么通过最近的收到的ACK判断已经发送的segment是否loss。具体来说一个segment被认为loss需要满足两个条件：

- 比这个segment之后的segment已经被ACK了
- 自这个segment 发送已经超过了一段时间，这个时间阈值设置为当前srtt + reordering window（因此sender需要记录segment的发送时间）

RACK优于传统快重传的地方就在于收到ACK feeback后是通过时间阈值来判定loss，而非duplicate  counting，从而能够在一些乱序程度高的链路中依旧有效（不会导致虚假重传）。

那么这个时间阈值怎么确定呢？为了能够容忍乱序，RFC8985中也给出了一个动态调整reordering window

值的算法，调整的思想是

- reordering window从一个small fraction of RTT的初始值开始，如果链路上没有观察到过reorder，那么从0开始
- 如果收到了对端的DSACK（说明有虚假重传），表示reordering window设置的小了，需要调大window值
- reordering window值需要以当前srtt为上界。



#### TLP

RACK依赖于ACK feedback来判定loss，但是有些情况下可能没有足够的ACK feedback，比如tail loss。TLP通过主动发送probe包来引出对端的ACK来 解决这个问题。TLP部分的核心有两个：

- 什么时候发送probe
- probe内容是什么

第一个问题通过设置一个probe timer（PTO）来解决，定时器到期时如果没有ACK则触发发送一个probe来引出ACK/SACK feedback。PTO的默认值是2*srtt（具体也会根据一些条件来调整）。加上前面的RACK reordering timer、传统的PTO timer，现在TCP每个连接上将会有三个timer，由于这三个timer同时只会有一个生效，实现上可以通过复用一个timer来管理这三个timer。

第二个问题probe的内容是什么。协议中规定probe可以是是下一个新的segment或者重传当前flight里最后一个segment。



OK，以上就是RACK-TLP丢包检测算法的概括性介绍，具体实现可以参考RFC8985。TCP重传相关内容回顾到次为止，下面进入正题。



## QUIC ACK机制

先定义几个概念：

- Ack-eliciting frames:  除了ACK、PADDING、CONNECTION_CLOSE帧之外的其他帧
- Ack-eliciting packets: 包含Ack-eliciting frame的packet，接收到ack-eliciting packet的一端需要在承诺的最大ack delay时间内回应ack，因此称为ack-eliciting
- In-flight packet: 很容易理解，就是发出去但还没有被ack、宣告丢失、丢弃等packet



### 什么时候需要发送ACK

前面提到QUIC 用packet number来唯一标识packet，并且PN在所处的PN space中是单调递增的，在packet丢失导致重传的时候，QUIC将重新构建新packet，选择性重传必要的数据，因此QUIC避免了TCP的重传歧义问题。和TCP一样，QUIC通过接ACK机制来保证packet的可靠性，并且QUIC对什么时候要进行ACK做了一些规定：

- 所有的packet 都需要被ACK一次或者以上，这点就和TCP不一样了，TCP中seq 段长度为0的segment（比如单纯的ACK segment）是不需要对端ACK的
- ack-eliciting packet需要对端在承诺的max ack delay前进行ACK。其中Initial 和Handshake 阶段的ack-eliciting packet需要在收到后并且能ACK时立即ACK，0-rtt和1-rtt阶段的packet则允许在max ack delay时间内ACK。
- non-ack-eliciting packet则无需对端在规定时间内特意ACK，而是只能随着对端的ack-eliciting packet一起的通过ACK frame来ACK。这点很重要，意味这一端不能用一个non-ack-eliciting packet（携带ACK frame）去响应对端的non-ack-elicing packet，否则就会出现双端进行non-ack-eliciting包的无限循环了。

上面几点是QUIC强制所有实现都需要的实现的行为（MUST），除此之外QUIC还规定了一些协议认为应该要实现的立即产生ACK行为（SHOULD）：

- 当收到一个乱序的packet时（PN出现gap），这和TCP 收到乱序的包要立即ACK原理类似。
- 当收到packet的IP header里包含ECN标识，此时表示网络上明确出现拥塞，也需要立即ACK，以便让对端更快响应拥塞事件



### ACK频率的 tradeoff

产生ACK的频率是一个性能的tradeoff，及时、高频的ACK有助于对端检测loss、及时重传、预估RTT、识别拥塞等操作，但是会增加双端的计算负载和网络带宽。QUIC建议至少收到两个ack-eliciting packet再去ACK，这一建议也来源于TCP的一个通用的建议，如果有网络条件、对端CC或者其他一些先验知识的话，可以采用其他有效的策略。



### 管理ACK ranges

和 TCP的SACK一样，QUIC的ACK  frame中包含收到的多个PN段，为了防止ACK packet丢失，PN段可能会和之前的ACK 中PN段有些重合。至于要包含多少PN段，QUIC没有强制规定，但是做了一些建议:

- ACK frame应当总是至少ACK最近收到的packet，这点QUIC给出的解释是在乱序达到的情况下最近收到的packet信息需要及时通过ACK反馈给发送端
- ACK  frame中也应当包含当前收到的最大的PN。
- ACK frame应当要去fit within当前的QUIC packet，如果不能，就去掉老的PN段（PN最小的那些）。
- 接收端一般都会维护一个待ACK的收到的PN集合。 在需要ACK时基于这个PN集合来生产Ack frame中的PN段，如果后面收到了对端对ACK的ACK，那么应当从待ACK PN集合中删掉对应的PN段。



### ACK delay测量和上报

每个ACK frame里会有个ACK delay字段用于显式上报接收端故意引入的delay，这个delay衡量的是从收到ACK frame中最大的PN对应packet开始到ACK发生时为止。

接收端主动上报ack delay有助于发送端测量path rtt。





## QUIC 丢包检测/重传机制



QUIC的丢包检测非常类似于RACK-TLP，一样都是基于ACK 来检测是否丢包，利用PTO来确保ACK被收到。一旦检测到丢包就需要进行重传来恢复。注意和TCP不同的是QUIC里没有和TCP一样的RTO超时重传packet机制。



### 基于ACK的丢包检测

基于ACK的丢包检测总的来说就是发送端通过ACK的顺序结构检测到了可能有丢包产生，满足下面条件就会认为是丢包：

- 在当前packet之后的发送的packet已经被ack了，并且它比被ack的 packet早了kPacketThreshold个包（packet threshold）
- 在当前packet之后的发送的packet已经被ack了，并且距离当前packet发送时间已经过了一段时间了 (time threshold)，这个时间设置为

```text
max(kTimeThreshold * max(smoothed_rtt, latest_rtt), kGranularity)
// 其中kTimeThreshold推荐为9/8，效果就是比当前smoothed_rtt大一点。

```

    实现上time threshold通常依赖一个timer，发送端在收到一个ack后（假设其ack了当前packet之后发送的packet），如果当前packet还没有被认为丢失，那么会设置一个丢包检测的timer，timer在当前packet sent time + time threshold 时到期，到期时将触发重传。



### PTO超时

QUIC的Probe timeout 是为了能够在发送了**ack-eliciting**包后一段时间没有收到ack的话触发发送一个或者两个probe包 。PTO是为了能让QUIC连接能够从丢失tail packet、或者对端ack丢失的情况下恢复。发生了PTO超时不意味着一定有丢包，发送端也不能因此就将PTO 超时的packet标记为丢失，而是根据收到的ACK进行丢包检测。



PTO时间通过下面方式计算：

```text
PTO = smoothed_rtt + max(4*rttvar, kGranularity) + max_ack_delay
```

- 对于Initial、 Handshake阶段的packet，在设置PTO时候max_ack_delay值设置为0，因此对端对于这些packet不会delay ack。
- 对于application data packet的PTO timer只能在连接完成handshake 确认后才能设置，不然发了probe，对端可能没法ack。
- 当发送端有新的ack-eliciting packet发送时重启PTO timer，这和TCP的timer一样，一个连接上只会有一个PTO timer。
- 如果一个packet达到PTO后，QUIC发送probe，然后PTO值本身需要进行指数backoff（和TCP的RTO一样）。当收到ack后PTO重置。
- 如果当前连接上有前面提到的用于丢包检测的timer时，不允许设置PTO timer



### RTO Probe

当RTO超时时，QUIC规定要发送一个或者两个probe，去elicit对端的ack。那么问题来了，这个probe packet应该包含什么呢？总的来说QUIC并没有明确规定probe里应该包含什么，只是说应当在probe packet里发送新数据，如果连接上没有新数据的话也可以根据应用层策略去重传之前的数据。另外，**如果想尽快elicit出对赌的ack的话，还可以将probe packet的PN故意跳过一个，这样对端收到后会立即ack。**



## QUIC RTT预估

TCP的预估的RTT直接用于设置RTO，因此会包含对端的主观delay。QUIC里则将客观的链路RTT和主观delay分开，预估的的smothed_rtt是纯粹的链路RTT。

QUIC packet的PN单调递增，因此根源上就不会出现重传歧义。QUIC的sender在收到一个携带ack frame的packet后也同样面临两个问题：1） ack frame里通常ack了sender 的 多个packet，因此sender需要决定用哪个packet的send_time作为rtt的起始时间。这个问题在TCP里 是由receiver决定的。2)  和TCP一样，sender也不是收到一个带ack frame的packet就产生一个rtt采样。

- 第一个问题：sender在收到一个ack frame后，用ack frame里ack的PN最大的包的send_time作为rtt的起始时间，拿当前时间减去这个send_time作为一个rtt的样本(latest_rtt)，但是endpoint可能会delay ack，QUIC会在ack frame里携带自己对最大PN的主观delay。因此实际链路的RTT （adjust_rtt）是latest_rtt - delay。
- 第二个问题：QUIC规定只有ack frame在第一次ack最新的PN时，这个ack才会拿去做RTT采样。（和TCP类似）



有了rtt采样(latest_rtt)后，和TCP 一样，需要计算平滑rtt

```text
ack_delay = decoded ack delay from ack frame
if (hanshake  confirmed)
   ack_delay = min (ack_delay, max_ack_delay)
 
adjusted_rtt = latest_rtt
if (latest_rtt >= min_rtt + ack_delay) :
   adjusted_rtt = latest_rtt - ack_delay

smoothed_rtt = 7/8 * smoothed_rtt + 1/8 * adjusted_rtt
rttvar = 3/4 * rttvar + 1/4 * abs(smoothed_rtt - adjusted_rtt)
```



- 可以看到其中引入了一个新的min_rtt：min_rtt是度量的一段时间内观测的链路上最小的latest_rtt (包括ack_delay的rtt)。在上述rtt预估算法中min_rtt充当了adjusted_rtt的下界（如果adjusted_rtt < min_rtt，那么不用adjust，直接用latest_rtt作为样本），限制adjusted_rtt是为了防止对端误报ack_delay。
- 对端上报的ack_delay可能大于承诺的max_ack_delay，超出的部分QUIC将其归因为对端非故意引起的delay，而且很可能重复发生，因此需要作为path delay的一部分，需要纳入到path rtt计算仿作，不应该从latest_rtt中减去，所以对ack_delay有个截断操作。





## QUIC ACK/重传机制总结

- ACK机制
    - 什么时候要/不要ACK
    - 怎么ACK （怎么构建ACK frame）
- 重传机制
    - QUIC怎么检测丢包
        - packet threshold
        - time threshold
    - PTO超时机制
        - 什么发送PTO probe
        - 怎么构建probe packet
    - RTT预估







