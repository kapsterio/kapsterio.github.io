---
layout: post
title: "Deflate Algorithm"
description: ""
category: 
tags: []
---

## 背景
前一篇[blog](http://kapsterio.github.io/test/2021/07/19/story-of-deflate.html)中，为了弄清楚长久一来一直困惑我的deflate、zip、gzip、zlib这些关系，我在互联网的各个角落里翻阅了不少RFC和web pages，我觉得算是基本弄清楚它们之间来龙去脉和联系了，接下来进入正题，写写关于Deflate算法。我对这个算法的兴趣源自于之前在公司内部做的一个优化imo客户端和服务端通讯协议相关的工作，是的，我们代码在对通信协议数据使用的zlib对数据进行的压缩，在深入去了解deflate算法原理和zlib使用后，我对原来的代码做出了一些比较有效的优化。所以我觉得有必要去写一点Deflate算法的东西记录下。


在学习Deflate时候，我先是找到了[RFC 1950](https://datatracker.ietf.org/doc/html/rfc1951)也就是Deflate压缩标准来啃，通读一篇发现还是对deflate的工作原理和细节还是模模糊糊，原因是RFC 1950里只有标准，不涉及解释，也就说只说了怎么做，没有解释为什么这么做。后来发现zlib的作者在[An Explanation of the Deflate Algorithm](https://zlib.net/feldspar.html)一文中对Deflate算法的原理和trouble points做了比较全面的解释，但是我第一次阅读这篇blog时对后半部分基本上没怎么看懂，原因是一方面我水平菜，另一方面我觉得是Antaeus Feldspar大神在这篇blog里省略了很多细节。在各种搜索了解这些缺失但是对理解算法很有必要的细节后，再读起来就比较顺畅了。这篇blog的内容就是扩充下[An Explanation of the Deflate Algorithm](https://zlib.net/feldspar.html)的内容，补齐其缺失的细节。

## 前置知识
> It's important before trying to understand DEFLATE to understand the other two compression strategies that make it up -- Huffman coding and LZ77 compression.

Deflate算法主要基于前人的两个算法，LZ77压缩算法和Huffman coding。

### LZ77
这里先简单介绍下LZ77算法，基本思想是数据中存在很多重复出现的字符串（也可以是字节流），重复的越多，可压缩的空间越大。对于这些重复出现的字符串，可以用<length, distance>  pair来表示，distance是到上一次这个字符串出现位置的距离，length是这个重复串的长度。

这里面有几个细节：
- 重复串搜索只会在往前的一定长度范围内由近到远搜素，LZ77称之为滑动窗口(Search Buffer)，一般这个窗口大小是32K，滑动窗口大小决定了distance的取值范围。
- 除了前向用于搜索重复串的Search buffer外，还有个后向的Look-ahead buffer，作为当前待编码的字符串区，因此寻找重复串的过程就是在search buffer里查找和当前look-ahead buffer最长的匹配字符串（常见的动态规划问题）。look-ahead buffer大小决定了length的取值范围。
- deflate里认为只有重复串长度超过3个字节，才采用这种表示方法，同时最多允许匹配258个字符，因此length的取值范围是3-258（256个不同的值），这个限制是在压缩率和算法计算复杂度间做权衡。


encoding的伪代码如下：
```
while look-ahead buffer is not empty 
    go backwards in search buffer to find longest match of the look-ahead buffer 
    if match found 
        print: (offset from window boundary, length of match); 
        shift window by length; 
    else 
        print: first symbol in look-ahead buffer;
        shift window by 1;
    fi
end while
```

可以看出encoding过程需要做个最长匹配字符串的查找，也可以选择合适的数据结构来表示search buffer/look-ahead buffer来加速查找。encoding的输出是个token stream，token可以是<length, distance> 也可以是symbol本身。

decoding的伪代码：
```
for each token (<length, distance>  | symbol) 
    if token is symbol 
        then print symbol; 
    else 
        go reverse in previous output by offset characters and copy character wise for length symbols; print symbol; 
    fi 
next
```


### Huffman coding
哈夫曼编码应该不需要多做介绍了，每个学过数据结构的都具备这个知识。recap下Huffman coding是一种前缀编码，根据统计每个字符在整个待编码数据中出现的次数，从小到大排序，由底向上动态构建一颗二叉树，二叉树的叶子节点就是每个字符，出现次数越小的字符处于二叉树的底部，对应的码字的长度也越长。这样构建的二叉树就是haffman树，整个字符集以及其对应的码字构成整个码表。由于码表是根据要编码的数据动态生成的，想要解码必须要有这个码表，因此码表需要和数据一起传输。

需要说明的是同一个数据集在经典的haffman算法下，可以有多棵haffman树存在，因为节点位于二叉树的左边还是右边对最终的编码长度来说没有影响，有影响的只是节点位于树的哪层（也就是字符在被编码后的bit长度，也被称为code length）。可以这么说：对于如果两个码表中码字的code length一致的话，那么这两个码表最终的编码效果也是一致的。因此，在deflate算法中，为了最大化减少传输码表所占用的数据量，在构建haffman树时对树的形状做了两个限制，使得同一个数据集唯一生成一颗haffman树，与此同时，在传输码表的时候只需要传输字符和其对应的code length就要可以了。 这两个限制也很简单：

- 当叶子节点和一个中间节点构建二叉树时将叶子节点放在左边，中间节点放右边。
- 当两个叶子节点构建二叉树时，将先出现的字符的节点放左边，后出现的放右边。

这样构建出的二叉树越靠左边树越浅，越靠右边树越深，极为不平衡，特殊且唯一。



### Deflate中LZ77和Haffman coding怎么结合起来

总的来说，Deflate先应用LZ77压缩策略对原始数据进行压缩，得到<length, distance> | literal 流，然后应用haffman coding对 distance、length、literal分别进行编码得到最终的压缩数据流。

应用LZ77的过程没啥好说的了，比较trick的是应用haffman coding的过程。

#### 对distance的haffman coding
前面我们知道distance的取值范围取决于search buffer 的大小，32K的search buffer对于distance的取值范围为1-32768，也就是说distance码表中最多可能有32768个码表，我们知道haffman树的高度在最坏情况下渐近于码表大小。要知道Deflate诞生于上世纪90年代，那个以KB计算内存的时代内存是非常宝贵的，直接应用经典haffman算法构建haffman树时对内存的消耗、计算量的要求很容易就超出当时的硬件条件。因此Deflate算法做了个很牛逼的优化——将distance的value space划分为30个变长的段，然后对这30个段进行haffman coding，段内根据段区间大小进行等长编码。具体如下：
![distance coding](/public/fig/distance_coding.png)

其中code是段的编号，bits表示该段的区间长度。可以看出，distance 1,2,3,4这四个特殊的distance没有划分，一个distance值占用一个段，distance值越大，段划分的越稀疏，这样做的理由是考虑到重复串出现的空间局部性。举个具体的例子，假设17-24这个distance段的huffman coding是二进制的110，那么distance 17-24最终的coding为：
```
17-->110 000

18-->110 001

19-->110 010

20-->110 011

21-->110 100

22-->110 101

23-->110 110

24-->110 111
```
Deflate通过这样的方式在码表长度和算法的计算复杂度之间做了trade off


#### 对literal/length 的haffman coding
literal就是原始数据中的一个个字符（字节），所以一个literal的取值范围是0-255（256种不同的取值）。length前面说了，取值范围是3-258（也是256个不同的取值）。Deflate在这里比较trick地将literal值、length值、以及数据块结束标识全部统一到一个alphabet中进行编码，得到也是同一个码表。这样做的好处是不需要在最终的编码中用一个标识位来标识当前编码的类型（是literal还是length或者是块结束标识）。当然也有代价，代价就是huffman树变大了。Deflate对literal取值分布没有做任何假设，但对length的取值做了和distance类似的假设，即length越大意味这字符串匹配的长度越长，出现的概率越小，因此Deflate对length的value space做了类似distance的分段处理，将256种取值划分为29个区间段。如下表所示：

![length coding](/public/fig/distance_coding.png)

细心的读者会发现，285这个length的bits是0，意味着285作为一个独立的段参与haffman coding中，为什么？我猜测是由于285作为Deflate规定的最大匹配串长度，也就说所有超过285的匹配字符串长度都被截断到285了，因此其出现的概率会相对高些。

OK，这里暂停总结下目前为止Defate算法的编码过程：
首先应用LZ77算法对原始数据进行压缩，生成(<length, distance> | literal)数据流，然后对数据流应用huffman coding算法进行统计分析，生成了两个haffman码表，一个是literal/length构成的alphabet所生成的码表（简称为码表1)，一个是distance值构成的alphabet所生成的码表（简称为码表2），再由这两个码表对(<length, distance> | literal)数据流进行编码得到最终的数据码流。

解码过程就是上面的逆过程，假设已经重建好码表1和码表2，首先应用码表1对数据流进行解码，如果发现是literal则直接输出，如果是length，继续用码表2对length后面的distance进行解码，得到<length, distance> pair，最终将生成的(<length, distance> | literal) 数据流交给LZ77解码得到原始数据。进度条到这里已经过半了，但是故事还没有结束，前面说过码表本身需要和压缩的数据流一起来传输/存储，下面继续Deflate算法对码表本身的处理。


#### 对huffman码表本身的encoding
前面提到，Deflate选择构建一颗最特殊的huffman树（最左倾的树）来进行huffman编码，这样码表可以用字符以及其对应的code length来唯一表示，大大缩小了码表的数据量。以literal/length码表为例，整个alphabet大小是286（256个literal + 1个块结束标识 + 29个length段），因此整个码表可以用长度为286个code length序列来表示，每个code length唯一对应一个alphabet中的字符。Distance码表也一样，可以用一个长度为30的code length序列唯一表示。通常来说这两个code length序列越往后code length值为0的可能性就越大，0意味着alphabet中没有对应的元素（字符）。所以在这里Deflate用了一个小的trick，它将这两个code length序列trim一下，把trailing的那些0都去掉，然后把他们合并称一个code length序列。由于事先知道这两个序列的原始长度，根据经过trim再合并得到的这个code length完全能还原出原始的两个code length序列。

可以看出这个序列中依旧会出现很多连续的0，或者1等等code length。

因此Deflate意识到这点，先对这个code length序列做一次[run length encoding](https://zh.wikipedia.org/zh-tw/%E6%B8%B8%E7%A8%8B%E7%BC%96%E7%A0%81)，也就是游程编码。