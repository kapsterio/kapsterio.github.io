---
layout: post
title: "nettt WeightedFairQueueByteDistributor算法和实现"
description: ""
category: 
tags: []
---
# Overview

这篇blog将具体介绍netty中对http2协议中stream dependency部分的实现，分为两部分，第一部分先介绍下netty http2所参考的[HTTP/2 priority implementation in nghttp2
](https://docs.google.com/presentation/d/1x3kWQncccrIL8OQmU1ERTJeq7F3rPwomqJ_wzcChvy8/edit#slide=id.p)采用的算法，第二部分分析下netty http2中WeightedFairQueueByteDistributor的算法和具体实现。


# HTTP/2 priority implementation in nghttp2

## 问题是什么
http2中stream会依赖父stream，父stream优先于子stream发送数据，当父stream不能proceed时候，父stream的分配资源被所有子stream按照所占的权重来共享分配。所有stream形成一颗自顶向下的依赖树，树的根结点是stream 0，算法要解决的问题是将本轮可以发送的数据量在这颗依赖树上所有active streams间进行分配。

有个前提：分配数据量有个最小单元，也称为chunk，这个前提也很容易理解，这个chunk大小的选择要做到效率和公平间平衡，选择的太小的话，效率低，太大的话，不公平。

在前面的UniformStreamByteDistributor中，chunkSize是根据当前最大可写的数据量maxBytes 除以当前active stream数动态计算得到。在这里chunk大小的选择将是影响算法表现的一个重要因素。

## 基本思想

![flat case](/public/fig/basic.png)

图示中chunk size为100B，分配模式如图所示，一次stream5，跟着两次stream7。


当应用到子树上时：左子树（#3）和右子树(#9)还是以，一次左，两次右的分配模式出现，由于左子树还有内部节点，所以每次分配左子树的时候需要在整个子树上按照同样算法分配（即以chunk size为分配单元，把分配单元按比例依次给到#5和#7）。

整个思路非常类似于操作系统里进程调度思想，把时间片在整个进程树上进行分配。是RR算法的一个很自然的带权重扩展——[WFQ](https://en.wikipedia.org/wiki/Weighted_fair_queueing)

![nested case](/public/fig/nested.png)



## 实现思路

### 数据结构 —— prority queue per stream

算法为每个stream维护一个优先级队列来维护当前stream的active的子stream的调度优先级（注意：如果一个stream自己不是active，但是以他为根的子树中存在active的stream，那么这个stream也会在其父stream的优先级队列里）。这里的调度优先级就是实现意义上的调度优先级，当前优先级高的将被优先调度，和http2的priority无关。具体实现上通过定义一个pseduo time来表征优先级，值越小标识优先级越高。 

OK, 有了per stream 的优先级队列后，决定每个chunk应该分配给哪个stream的调度算法的大致过程则是：
- 从树根（stream 0）开始调度
- 如果当前stream有数据要发送，直接将chunk分配给当前stream
- 如果当前stream没有数据发送，并且当前stream的优先级队列也是空的，退出本次调度
- 如果当前stream没有数据发送，优先级队列不为空，则`pop`出队列里最高优先级的stream，递归调度这个子stream。
- 递归调度完子stream后，如果子stream或者以他为根的子树中还存在active stream，则将对子stream`更新pseduo time`，重新`push`入当前stream的优先级队列中。
- 本次调度结束

要实现前面的调度思想，很自然得出这样的递归调度的思路，现在问题只剩下怎么计算stream的pseduo time，来达到`flat case`图中的调度效果，可以看出优先级应该和stream的weight以及被调度的次数有关：和weight成反比，和调度次数呈正比。
因此一种计算方式是： priority = C * n / w， C是一个常量，n是当前stream被调度的次数，w是当前stream的weight。

以`flat case`为例，初始状态下: #5和#7都是0，所以先调度一次#5，之后更新#5的优先级为C * 1 / 1 = C，因此第二次调度的#7，之后更新#7的优先级为 C * 1 / 2 = 1/2 * C，因此第三次调度的还是#7，之后#7的优先级则更新为 C * 2 / 2 = C。OK，再调度的话则是#5，如此往复，看起来挺简单美好也符合调度思路。但是，问题来了，假设在#5和#7之间按照权重轮流调度一段时间后，#0下面新增了一个权重为1的stream #8，由于调度次数为0，#8将会一直被调度直到达到和#5的次数后再正常的按照权重调度，这就破坏了权重的含义（使得更高权重的#7一直得不到调度）。

这就有了第二思路： 为每个优先级队列维护一个优先级的base，所有入队的stream的优先级计算都以这个base为基础，增加一个反比于weight的项，然后每次调度pop出一个stream后以这个被pop出的stream的priority来更新这个base，起到当一个stream被调度后再次入队的时候priority值和考虑调度次数时一致，另一方面，有新的stream进来的时候由于需要以base为基础，得到的priority值不至于太小而被一直优先调度。因此可以得出计算priority (pseduo time)的公式为：
```
t[i] = t_last[p] + nsent[i] / weight[i] * K
```

- t[i]是stream #i的pseduo time，
- p是i的父stream。
- t_last[p]：是stream p的优先级队列中最后一次pop 出来的stream的pseduo time
- nsent[i]：是以stream #i为根的子树上一次分配到的字节数 （这里没有使用常量C的原因是，stream #i上一次不一定实际分配了C个字节，可能由于stream的发送窗口限制了，另外需要特别注意的是，考虑到nested case，这里是以#i 为根的子树上一次分配的字节数，不是#i自身分配的字节数）
- weight[i]: 是stream #i的weight
- K是一个常量，用来补偿整数除法导致的精度损失 （通常为256）

因此，调度算法用伪代码表述为：（PPT中算法不太正确，因为nsent[i]在PPT中为stream #i本身上一次实际分配的字节数，但是例子中又是以stream #i为根的子树上一次分配的字节数）

```

Always start from root stream #0

def schedule(p):  //返回以p为根的子树在本次调度中实际分配的字节数
  if stream #p is active
    send data for #p
    return atual sent data size 
  if #p’s queue is empty:
    return 0
  pop #i from queue
  update t_last[p] = t[i]
  update nsent[i] = schedule(i)
  if #i or its descendant is “active”:
      update t[i] and push it into queue again
  return nsent[i]

schedule(0)
```

需要注意的是这个算法里说的`active` stream等价于stream有数据要发送，优先级队列里维护的stream也都是`active`的。

# WeightedFairQueueByteDistributor实现

## 数据结构

### State 与 State树
`Http2Stream` 接口里没有体现Stream间的依赖关系，依赖树结构关系、分配调度所需要的状态数据都被放到了WeightedFairQueueByteDistributor内部实现的State中，每个Http2Stream会对应一个State。State主要有以下属性
- final int streamId: state所标识的stream id
- Http2Stream stream: state所关联的Http2Stream对象，可能是null（标识这个stream还没有被创建或者已经销毁，不是active）
- State parent: 父Stream 的state
- IntObjectMap<State> children: 所有依赖当前stream的子stream的state，是一个map，key为stream id，value是stream的state
- int streamableBytes: 当前stream上可以被调度的字节数
- int dependencyTreeDepth:  当前stream 所处在树的深度
- int activeCountForTree: 以当前节点为根的state子树上有多少个active的stream，作用是什么？？？
- final PriorityQueue<State> pseudoTimeQueue: 重要！这是上一节算法中定义的per stream的优先级队列，按照优先级存放当前stream的所有子active streams的state ?
- long pseudoTimeToWrite: 重要！这类比于上一节算法中的t，也就是pseudoTimeQueue优先级队列中优先级排序依据
- long pseudoTime: 重要！这类比于上一节算法中的t_last, 也就是当前进入当前stream的优先级队列时计算优先级所需要的base
- long weight: 就是当前stream依赖其父stream的权重
- long totalQueuedWeights: 当前stream所有子stream的权重之和
- byte flags: 一个字节的标记位


### WeightedFairQueueByteDistributor实例的数据结构
`WeightedFairQueueByteDistributor`实例本身是连接Connection维度的，他的数据结构也都是Connection维度的，主要有：
- final IntObjectMap<State> stateOnlyMap: http2协议中允许在一个不存在的(idle/closed)stream上发送PRIORITY帧，也允许stream去depend on一个不存在的stream，同时http2还建议即便一个stream close后，还要继续保留stream的Prioritization State一段时间（以避免由于对子stream的重新Prioritization而可能导致的Prioritization信息损失）。基于以上情况，需要有个管理这些没有Http2Stream对象的State，stateOnlyMap就是这个作用，key是stream id, value是state对象。
- final PriorityQueue<State> stateOnlyRemovalQueue: 这个queue的作用是按照自定义的优先级来维护这些没有关联Http2Stream的state。
- State connectionState: state树的root


### 对State树和non-active state map结构的维护
定义了数据结构之后，剩下的操作就比较不言自明了，主要是对state树进行非常old-school的树操作，这里抽了几个方法，简单介绍下实现和调用时机。


#### onStreamAdded
这个回调发生在stream被创建的时候，那么stream创建具体有哪些时机？
- 收到对端发的HEADER帧，此时stream处于open状态
- 向对端发送HEADER帧, stream也是open状态
- 客户端在进行clear text http1 upgrade成功后，协议规定要reserve id为1的stream标识当前的upgrade请求，对于服务器端也需要做类似的事情，对upgrade请求的response也需要发送在stream 1上， 此时stream处于half close状态，
- 服务器端发送Push promise帧，此时stream处于reserved状态

此时，存在两种情况，1）stateOnlyMap中已经有了相应的state，那么需要从stateOnlyMap和stateOnlyRemovalQueue删除这个state，然后将state关联上stream对象。2）stateOnlyMap中没有这个state, 说明这个stream id之前没有出现过，那么需要初始化state的一些数据结构（建立和connection stream的state的父子关系），加入state树



#### onStreamRemoved
当stream/connection被close时，这个回调得到执行。此时需要的做的是，首先解除state和stream的关联，然后将state加入stateOnlyMap和stateOnlyRemovalQueue中，如果stateOnlyRemovalQueue的size超过最大限制了，需要将queue中优先级最低的出队，然后将这个出队的state从state树中删除，再从stateOnlyMap中删除。（从state树中删除会涉及到很多数据结构的变更，比如当前state的和其parent的父子关系、当前state的如果还有子的话，需要将所有子state加到parent的children集合中，还可能会加入到新的parent的pseudoTimeQueue中）


#### updateDependencyTree

```
        if (newParent != state.parent || (exclusive && newParent.children.size() != 1)) {
            final List<ParentChangedEvent> events;
            if (newParent.isDescendantOf(state)) {
                events = new ArrayList<ParentChangedEvent>(2 + (exclusive ? newParent.children.size() : 0));
                state.parent.takeChild(newParent, false, events);
            } else {
                events = new ArrayList<ParentChangedEvent>(1 + (exclusive ? newParent.children.size() : 0));
            }
            newParent.takeChild(state, exclusive, events);
            notifyParentChanged(events);
        }
```
核心逻辑是这段代码，实现了“如果变更一个stream的依赖去依赖自己的某个子stream，这个情况比较特殊，需要先把这个子stream改为依赖当前stream的之前的父stream，然后再变更当前stream的依赖” 这个操作。


#### updateStreamableBytes
这个接口的作用之前说了，是用来告诉Distributor，StreamState标识的流的Streamable（可写的字节)发生变化了。调用的时机是有数据发送前和完成数据写出后。这里很重要的是当stream有数据可发送时，需要更新state树上从当前state开始一直到state树根路径上所有state节点的activeCountForTree，如果activeCountForTree为0了，说明当前state没有数据可发送了，需要从parent的pseudoTimeQueue中移除，如果activeCountForTree之前为0，更新后不为0了，需要将state加入parent的pseudoTimeQueue中。这里将stream有数据可发送(`state.hasFrame() && state.windowSize() >= 0`)定义为active stream，pseudoTimeQueue维护的也都是active stream. 这里也可以看出State中activeCountForTree这个字段重要性，通过这个字段的值决定是否需要从pseudoTimeQueue中移除/重新加入。state树结构有变更时候activeCountChangeForTree方法都需要执行，递归传播到state树根。另外还有个字段
totalQueuedWeights基本上和pseudoTimeQueue维护的时机一致，因此totalQueuedWeights准确来说是state所有active 的直接子state的weight之和。


#### distribute算法

```

    private int distribute(int maxBytes, Writer writer, State state) throws Http2Exception {
        if (state.isActive()) {
            int nsent = min(maxBytes, state.streamableBytes);
            state.write(nsent, writer);
            if (nsent == 0 && maxBytes != 0) {
                // If a stream sends zero bytes, then we gave it a chance to write empty frames and it is now
                // considered inactive until the next call to updateStreamableBytes. This allows descendant streams to
                // be allocated bytes when the parent stream can't utilize them. This may be as a result of the
                // stream's flow control window being 0.
                state.updateStreamableBytes(state.streamableBytes, false);
            }
            return nsent;
        }

        return distributeToChildren(maxBytes, writer, state);
    }
    
    private int distributeToChildren(int maxBytes, Writer writer, State state) throws Http2Exception {
        long oldTotalQueuedWeights = state.totalQueuedWeights;
        State childState = state.pollPseudoTimeQueue();
        State nextChildState = state.peekPseudoTimeQueue();
        childState.setDistributing();
        try {
            assert nextChildState == null || nextChildState.pseudoTimeToWrite >= childState.pseudoTimeToWrite :
                "nextChildState[" + nextChildState.streamId + "].pseudoTime(" + nextChildState.pseudoTimeToWrite +
                ") < " + " childState[" + childState.streamId + "].pseudoTime(" + childState.pseudoTimeToWrite + ")";
            int nsent = distribute(nextChildState == null ? maxBytes :
                            min(maxBytes, (int) min((nextChildState.pseudoTimeToWrite - childState.pseudoTimeToWrite) *
                                               childState.weight / oldTotalQueuedWeights + allocationQuantum, MAX_VALUE)
                               ),
                               writer,
                               childState);
            state.pseudoTime += nsent;
            childState.updatePseudoTime(state, nsent, oldTotalQueuedWeights);
            return nsent;
        } finally {
            childState.unsetDistributing();
            // Do in finally to ensure the internal flags is not corrupted if an exception is thrown.
            // The offer operation is delayed until we unroll up the recursive stack, so we don't have to remove from
            // the priority pseudoTimeQueue due to a write operation.
            if (childState.activeCountForTree != 0) {
                state.offerPseudoTimeQueue(childState);
            }
        }
    }
```

distribute算法框架和上一节中算法类似，都是从树根(Connection stream)开始，递归选择一个stream进行分配，distribute方法返回也是这次分配实际分配的字节数，分配的算法框架也是先判断当前stream有没有数据要发送，如果有直接发送当前stream，如果没有，从PseudoTimeQueue中出队一个子stream，对这个子stream进行递归分配，然后更新当前stream的 PseudoTime的base，如果以子stream为根的子树上还有active的streams，需要再更新子stream的PseudoTime，并且将其重新加入当前stream的PseudoTimeQueue中。可以看出和上一节的算法框架基本一致。

区别于上一节算法有两个细节比较重要：
1) 怎么更新stream的PseudoTime (state的pseudoTimeToWrite字段)？
```
      //更新base
      state.pseudoTime += nsent;
      //更新pseduoTimeToWrite
      pseudoTimeToWrite = min(pseudoTimeToWrite, parentState.pseudoTime) + nsent * totalQueuedWeights / weight;
      
```
这里对pseudoTime也就是base的更新区别于上一节的算法，这里一个stream的base就是以这个stream为根的子树上所有发送字节数之和，新入队的stream将以这个值作为pseduoTimeToWrite。更新已分配stream的pseudoTimeToWrite时，通过数学归纳法可以很简单的证明pseudoTimeToWrite值一定比parentState.pseudoTime值小，所以上面的更新计算等价于：``` pseudoTimeToWrite += nsent * totalQueuedWeights / weight```。可以看出这里是采用了totalWeights作为K，```nsent * totalQueuedWeights / weight``` 可视作stream被调度一次后pseudoTimeToWrite的增量，同一个队列里不同stream增量的比值就是weight的比值。

2）每次distribute时最多分配的字节数怎么定？
在之前的算法中，每次分配的字节数为了简单起见假定是固定的常量，这样做问题就是这个常量的大小很难在效率和公平之间做权衡，选大了效率高但是不太公平，选小了虽然公平但是低效。netty这里做了个优化根据下面这个值动态决定这个大小：

```
min(maxBytes, (int) min((nextChildState.pseudoTimeToWrite - childState.pseudoTimeToWrite) * childState.weight / oldTotalQueuedWeights + allocationQuantum, MAX_VALUE)
```
这个值如果不看后面的allocationQuantum项的话， 就是(nextChildState.pseudoTimeToWrite - childState.pseudoTimeToWrite) * childState.weight / oldTotalQueuedWeights，将这项带入更新pseudoTimeToWrite的式子可以知道pseudoTimeToWrite的增量就是(nextChildState.pseudoTimeToWrite - childState.pseudoTimeToWrite)，因此这个算法的意图就比较明显了，就是当队列里下一个stream的pseudoTimeToWrite和当前stream的pseudoTimeToWrite相差较大时，多为当前stream分配一些字节数，因为分配少于这个值的话下一次分配还是当前stream，多做了一次分配工作，吞吐量就相对低些。至于allocationQuantum这个项作用也比较简单，就是当前一项很小的时，allocationQuantum能够保证不至于分配的太小而损失分配效率。


### 总结
简单总结下WeightedFairQueueByteDistributor的分配算法和工程实现:
- WeightedFairQueueByteDistributor 基于network scheduling领域内常见的WFQ调度思路，除了WFQ调度，还有FIFO、PQ、FQ、RR、WRR、DRR等等调度思路，对这些调度算法的一个概括性介绍可以参考[Queuing and Scheduling](http://what-when-how.com/qos-enabled-networks/advanced-queuing-topics-qos-enabled-networks-part-1/)，
- WeightedFairQueueByteDistributor实现参考了[HTTP/2 priority implementation in nghttp2
](https://docs.google.com/presentation/d/1x3kWQncccrIL8OQmU1ERTJeq7F3rPwomqJ_wzcChvy8/edit#slide=id.p)，并在pseudoTime计算、分配字节数计算等方面做了一些优化，取得更好的效率和公平兼顾
- WeightedFairQueueByteDistributor对http2协议中stream state Reprioritization部分基本都实现了，但是miss了一个point，就是当一个stream close时，他的children要成为这个closed stream的parent stream的children，此时这些children的weight按照协议时需要重新计算的（需要把closed stream的weight按照比例分配给children）。我在这个[PR](https://github.com/netty/netty/pull/11490)里补上了，