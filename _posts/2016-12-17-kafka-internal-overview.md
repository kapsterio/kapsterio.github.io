---
layout: post
title: "Kafka internal overview"
description: ""
category: 
tags: []
---
> WARNING: 本文完全来自Kafka的[官方文档](https://kafka.apache.org/documentation/#introduction)，结合自己的理解和提炼，属于二手知识，如果需要深入了解Kafka，建议直接阅读官方文档。主要关注两个部分：1）Kafka对外提供的行为和语义保证，2）Kafka的设计概要

<hr/>
# Introduction

## Kafka一些行为的要点：
kafka有个可配置的日志保留时间， 过了这个时间的日志，无论消费与否都被清除

消费者维护的仅有的metadata就是offset，非常轻量级，所以不用担心消费者很多，kafka处理不过来的问题。

<!--more-->

partition的作用就是为了scale和并行，将一个topic的日志分布在多个partition中，做到水平扩展。每个partition会有多个副本，分布在集群中不同的机器上，副本的数量可配置。每个partition有一个server充当这个partition的leader，其他server则充当followers，leader负责这个partition的所有的读写请求，follower作用仅仅是被动复制leader，仅在leader挂掉时自动进行选主，其中一个follower将成为新的leader。

生产者publish数据到指定的topic上，并且由它来负责将消息发往这个topic上的哪个partition，一般来说，可以使用round-robin方式来做简单的partition间的负载均衡，当然也可以按照一个特定的partition选择函数来确定发往的partition。

消费者有个消费者组的概念，消费组是订阅一个topic的最小单元，也就是说，如果所有的消费者实例在同一个消费组里，那么消息将在这些消费者实例间负载均衡。如果所有的消费者属于不同的消费组，那么消息将被广播到所有的消费者实例上。

那么在一个消费组内部，partition和消费者实例的关系是怎么样的呢？比如现在有4个partition,2个消费者实例，kafka实现的方式是每个消费者实例固定从两个partition中读取数据。事实上kafka动态维护一个partition和消费者的映射关系。当一个新的消费者实例加入这个消费组时或者一个消费者从消费组中去掉时，这个映射关系也会动态的变化，伴随着partition的重分配。

kafka仅提供一个partition内部的有序性保证，一个topic内多个partition间则没有这个顺序保证。这种Per-partition的有序性保证结合生产者指定partition（通过partition函数）对于大部分应用来说是足够的，如果你想要做到全局有序，那么只能为这个topic配置一个partition了，这也意味着订阅这个topic的消费组中也只有一个消费者能工作。

## kafka的一些保证：
 - 对于单个生产者而言，它往某个特定的partition中sent的消息会按照sent的顺序被append到commit log中（被持久化）。意味着如果m1和m2是同一个生产者先后sent的，那么m1在这个partition log中的offset肯定比m2小。
 - 对于单个消费者而言，它看到消息的顺序是这个消息被存入partition log的顺序。
 - 对于一个有个N个副本的topic而言，kafka最多能容忍N-1个失败（不丢失消息）


## kafka vs 传统的MQ
- 传统的MQ往往可分为两种模型：queuing和pub-sub。queuing主要用来做scale，pub-sub则用于广播。kafka则通过消费者的概念将两者统一。

- kafka有着比传统MQ更强的顺序保证。传统mq将消息有序地存储在server上，如果多个消费者从这个队列中消费时，server会按照消息的存储顺序来hands out消息，然而当消息投递到多个消费者这个过程是异步时，消息到达消费者的顺序就乱了。这意味着消息的有序性因为并行消费的引入而被破坏了。传统MQ系统通常会通过“exclusive consumer”来work around，意味着同一个队列仅允许一个消费者。kafka的做法则好些，kafka做到per-partition的顺序性保证和消费组内部多个消费者之间的负载均衡。因为一个partition只会被消费组中的某一个消费者消费。另外，kafka中消费组中的消费者个数得小于partition数。

## kafka用作stream processing
kafka的stream api使得kafka起到和apache storm类似的功能。更多stream的内容见[文档](https://kafka.apache.org/documentation/streams)。

<hr/>
# Kafka的设计

## pagecache-centric design 
基于filesystem的数据结构实现线性存取，有效利用OS pagecache。采用这个设计的原因主要由首先：使用jvm内存有如下缺点：

- 内存开销很大，通常要double实际存储的数据空间
- gc是个问题

而基于filesystem有如下好处：

- 有效利用os的pagecache来利用主存空间，另外由于数据紧凑，可以cache住很多数据，并且没有gc的压力
- 即便服务重启了pagecache也还是热乎的，而放在jvm进程内存中则不一样，重启后需要重建整个数据结构
- 一句话: **`rather than maintain as much as possible in-memory and flush it all out to the filesystem in a panic when we run out of space, we invert that. All data is immediately written to a persistent log on the filesystem without necessarily flushing to disk.`** 即以pagecache为中心的设计，类似的设计还有varnish

## sendfile的应用
从disk到网络通常的数据transfer路径是这样的：

1. The operating system reads data from the disk into pagecache in kernel space
2. The application reads the data from kernel space into a user-space buffer
3. The application writes the data back into kernel space into a socket buffer
4. The operating system copies the data from the socket buffer to the NIC buffer where it is sent over the network

涉及两次系统调用和4次数据copy，sendfile则省略数据的中间拷贝过程，可以做到从pagecache直接发往NIC buffer，一次系统调用（什么？socket buffer都省了）。

pagecache和sendfile的组合使得通常消费者caught up大量消息时甚至都不需要kafka访问disk，整个消息集都通过pagecache直接发往网络到达消费者。

java中有关sendfile和zero-copy的内容见[这篇文章](https://www.ibm.com/developerworks/linux/library/j-zerocopy/)

## producer设计
- producer决定消息要发往哪个partition，partition的确定方式灵活，默认的round-robin方式来在partition间负载均衡，也可以指定一个partition function实现自定义的均衡方法。
- producer直接将数据发往某个partition的leader，没有路由层的介入，意味着producer端知道kafka有关所有partition的metadata信息。
- 可以为producer配置消息的批量send，异步send，以达到高吞吐

## consumer设计
kafka的消费者向partition的leader发起一个"fetch"请求，请求中指定offset，得到位置自这个offset开始的所有log。消费者拥有对这个消费位置有着非常重要的控制权（broker端只是保存消费者告知的消费位置）

### push vs pull
这点上kafka遵从传统的设计，即数据由producer push到broker，consumer从broker上pull数据消费。一些以logging-centric的系统比如 Scribe和Apache Flume，则采用的是push-based方式将数据push给下游。

这种pull-based的消费者的好处主要有：
- 轻量级，消费者不会因消息发送速率过高而被overwhelmed，即便落后了，也能在后面catch up上来
- 向消费者sent数据时可以做些激进的batching优化

## 消息delivery语义
分为两个问题：the durability guarantees for publishing a message and the guarantees when consuming a message.

对于第一个问题，kafka保证的是那些已经被committed进入log中的消息，能容忍副本数-1个节点的失败以至于不丢消息。同时kafka允许producer指定durability的级别，If the producer specifies that it wants to wait on the message being committed this can take on the order of 10 ms. However the producer can also specify that it wants to perform the send completely asynchronously or that it wants to wait only until the leader (but not necessarily the followers) have the message.

对于第二个问题，consumer的不同的做法有着不同的语义保证：
- 读完消息后，先保存position到broker，然后再处理消息。做到的at-most-once语义
- 读完消息后，先处理，成功后保存position到broker，做到at-least-once语义
- 引入two-phase commit协议保证exactly once

总的来说，kafka默认保证at-least-once语义。So effectively Kafka guarantees at-least-once delivery by default and allows the user to implement at most once delivery by disabling retries on the producer and committing its offset prior to processing a batch of messages. Exactly-once delivery requires co-operation with the destination storage system but Kafka provides the offset which makes implementing this straight-forward.


## replication
> A message is considered "committed" when all in sync replicas for that partition have applied it to their log. Only committed messages are ever given out to the consumer（注意这句话，只有committed的消息才会交给consumer）.This means that the consumer need not worry about potentially seeing a message that could be lost if the leader fails. Producers, on the other hand, have the option of either waiting for the message to be committed or not, depending on their preference for tradeoff between latency and durability. This preference is controlled by the acks setting that the producer uses. The guarantee that Kafka offers is that a committed message will not be lost, as long as there is at least one in sync replica alive, at all times.


### Replicated log
kafka的partition的核心就是一个replicated log。那么什么事replicated log呢？其实它是对一系列值的顺序达成一致性的处理过程的建模，就是分布式一致性协议所要解决的问题。最简单的实现方式就是：一个leader负责确定值和其顺序，其他的follower只是简单的copy leader达成的结果（raft算法就是这样的）。

如果leader不会挂掉的话，那就没有follower什么事了。当leader挂掉时我们需要从follower中选择一个新的leader，但是follower本身也可能落后leader很多或者挂掉，因为我们必须保证所选的是一个up-to-date的follower。因为这里就得出一个tradeoff：如果leader在宣称一条消息被commit前等待越多的follower，那么就会有越多的leader候选人，当然也就意味着更长的等待时间。

一个通常的做法对commit decision和leader election都采用majority vote（即过半，当然kafka并不是这样做的，但值得了解下）。具体来说：假设我们有2f+1个replicas，如果现在要求一条消息在被leader commit前必须有f+1个replicas收到了这条消息，此时我们选择一个新的leader只需要从任意f+1个followers中选择log最完整的那个follower即可，因为这个新的leader一定有所有的committed log（因为任意f+1个follower中至少有一个有up-to-date的replica），这也意味着可以容忍 (2f+1) - (f+1) = f个节点失败。

这种majority vote方法在commit decision时具有一个很好的性质：延迟取决于最快的那些节点。ZK的zab协议、raft、viewstamped replication等都是属于这一类。

majority vote的缺点在于，它可以容忍的失败节点数有限，意味着为了容忍一个节点失败集群中必须要有三份数据的拷贝；容忍两个失败则要求5份数据拷贝。经验表明在生成环境中只容忍一个节点失败是不够的，至少两个，但是对于大容量的数据存储系统而言，5份数据拷贝也是很不现实的，这也是为什么这种quorum算法通常用于集群配置管理（比如ZK）而不是数据分布式存储。比如在HDFS中，namenode的高可用基于majority-vote算法，而数据本身并不是。

kafka在选择commit decison的quorum set时所采取的方法有点不同，它动态地维护一个in-sync replicas集合（ISR），集合中所有的replica都catch up着leader。只有这个集合中的节点才能被选为leader。一个对partition的写操作只有当所有ISR中节点都收到后，才会被认为是committed。ISR集被持久化在ZK中。理想情况下，ISR集中有所有的follower，但只要ISR集中有一个节点，那么集群就被认为是可用的（其他节点都fail了）。因此，对于f+1个replicas而言，kafka可以容忍f个节点失败。相比majority vote而言，“延迟取决于最快的那些节点”这个性质就没有了。

### durability vs availability
假设我们运气不好，所有的replicas都fail了，而且不巧leader也挂了需要选主，此时将面临两个选择：

- 等待ISR集中的replica恢复过来，选择这个节点作为新的leader
- 选择最先恢复过来replica（不一定是ISR的）作为leader

这就是一个简单的consistency和availability的tradeoff。默认情况下，kafka采用的是第二个策略，这种行为被称之为“unclean leader election”。当然kafka提供配置禁用掉这个行为。

当durability的重要性高过availability时，可以采用如下两个topic-level的配置：

- disable掉unclean leader election
- 指定一个最小的ISR集大小，只要在ISR集大小大于这个值时kafka才提供写操作（这个配置需要和producer在ack级别是all才能真正起作用）

## replica management
上面关于Replicated log选主过程的讨论是仅针对单个topic的某个partition而言的，事实上，Kafka集群管理着成百上千个这样的partitions，分布在不同的broker节点上。也就是说集群中有很多的leaders，那么优化选主过程对可用性也是非常重要的。Kafka在集群的所有broker中选择一个broker作为“controller”，由这个controller在broker级别负责检测其他broker节点的fail情况，并负责给那些failed的broker上受到影响的partitions选主。这样做的结果就是所有选主操作都由controller负责，多个partition的leadership变动通知也可以做到批量发送，选主过程变得简单快速。如果controller挂掉了，那么剩下的broker节点将进行新的controller选举（即broker层面的选主）。