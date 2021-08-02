---
layout: post
title: "Huffman Codec in netty HPACK"
description: ""
category: 
tags: []
---
## Static Huffman Codec

前一篇blog[HPACK explained in detail](http://kapsterio.github.io/test/2021/07/29/hpack-explained.html)里，我试着分析了下HPACK的设计细节。其中HPACK对于string literal数据采用static Huffman coding来编码，这一篇blog将继续拓展这个话题，写写关于Hpack中static huffman decode/encode算法实现的内容，虽然HPACK里用的static huffman coding不涉及根据数据构造huffman树，但是想要实现一个高效的HPACK huffman encoder/decoder并不是一个non-trival的事情，尤其是将huffman 高效decoder的实现。接下来的内容分为两个部分，分别以netty HPACK的实现为例，介绍下目前工业界常见huffman decoder/encoder的算法。


<!--more-->

## HPACK Huffman encoder
Encoder部分比较简单，按个字节查static huffman码表，将byte encode成code，需要注意的有两点：
 - code是bits位数不是固定的，需要通过位运算将编码后得到的codes按字节依次写入最终的流里
 - HPACK中string literal representation类型数据的格式是flag + length + data + padding，其中flag标识数据是否是huffman encoded，length是编码后数据的长度，data是编码后实际的数据。问题来了，length在实际huffman encode前是未知的，所以encoder的实现常见有两种做法：1）分配一个临时buffer用于暂存编码数据，完成后得到length，将length的integer representation编码写入最终流、然后再将临时buffer中的encoded data copy到最终流。2) 先遍历原始数据一遍基于huffman 码表得到最终编码的length，写入最终流，再遍历一遍原始数据编码得到data写入最终流，这种做法需要对原始数据的遍历。netty中就是这么做的。

 Netty HPACK encode String literal的代码如下：

 ```
   private void encodeStringLiteral(ByteBuf out, CharSequence string) {
        int huffmanLength;
        if (string.length() >= huffCodeThreshold
                && (huffmanLength = hpackHuffmanEncoder.getEncodedLength(string)) < string.length()) {
            encodeInteger(out, 0x80, 7, huffmanLength);
            hpackHuffmanEncoder.encode(out, string);
        } else {
            encodeInteger(out, 0x00, 7, string.length());
            if (string instanceof AsciiString) {
                // Fast-path
                AsciiString asciiString = (AsciiString) string;
                out.writeBytes(asciiString.array(), asciiString.arrayOffset(), asciiString.length());
            } else {
                // Only ASCII is allowed in http2 headers, so its fine to use this.
                // https://tools.ietf.org/html/rfc7540#section-8.1.2
                out.writeCharSequence(string, CharsetUtil.ISO_8859_1);
            }
        }
    }
 ```
可以看出netty 对数据进行huffman编码的条件有两个：
- netty设置一个huffman编码的阈值，默认大小为512个字节，原始数据大小只有超过这个阈值，才会对其进行huffman encoding。
- 有时候huffman编码得到的数据大小可能要比原始数据还要大， 此时netty不会进一步对数据进行huffman编码。

计算huffman 编码数据长度的完整代码如下，其中lengths是static huffman 码表中的code lengths数组，元素是每个byte对于的code的bits位数。
```
    private int getEncodedLengthSlowPath(CharSequence data) {
        long len = 0;
        for (int i = 0; i < data.length(); i++) {
            len += lengths[data.charAt(i) & 0xFF];
        }
        return (int) ((len + 7) >> 3);
    }

```

对数据部分的huffman encoding完整代码如下：

```
    private void encodeSlowPath(ByteBuf out, CharSequence data) {
        long current = 0;
        int n = 0;

        for (int i = 0; i < data.length(); i++) {
            int b = data.charAt(i) & 0xFF;
            int code = codes[b];
            int nbits = lengths[b];

            current <<= nbits;
            current |= code;
            n += nbits;

            while (n >= 8) {
                n -= 8;
                out.writeByte((int) (current >> n));
            }
        }

        if (n > 0) {
            current <<= 8 - n;
            current |= 0xFF >>> n; // this should be EOS symbol
            out.writeByte((int) current);
        }
    }
```

可以看出，其用一个long类型的变量current寄存器存放code bits，用变量n当前current寄存器里还有多少bits没有写入数据流里。由于hpack的huffman编码最长不超过4个字节，每个byte数据编码后的code可以用一个int类型表示。nbits是当前byte编码后的bits位数。

```
current <<= nbits;
current |= code;
```
这两句代码将current左移nbits，将current的低位腾出来存放code，然后在while循环里从current的高位开始依次将每个byte输出到out。最后如果n不为0，说明current里还有n bits剩余，但是不满一个字节，因此需要padding到一个字节，写入out中。


### 优化
个人觉得HPACK的string literal中huffman编码这个先length再data的设计是个缺陷，为什么不利用EOS的huffman code呢？数据的末尾放一个huffman encoded EOS，这样解码解到EOS时候就停止就可以不用length字段了。


关于encoding，[Exploring HTTP/2 Header Compression](https://www.semanticscholar.org/paper/Exploring-HTTP%2F2-Header-Compression-Yamamoto-Tsujikawa/743aba87f0e3b58ba6b11237f51f1d06170125cb)这篇论文中还给出了一个优化的算法。算法利用了一个先验知识：就是huffman编码后的数据长度可以根据原始数据长度猜个大概，他们研究发现huffman编码平均能压缩20%的数据，因此压缩后的数据长度大概是0.8 * 原始数据长度。得到这个猜测数据大小后，就知道其integer representation所需的字节数了。比如7-prefix编码0-126只需要一个字节，127-254需要两个字节，255-16510则需要三个字节。如果猜数据大小范围没错的话，可以先空出来几个字节用于后面写入length，然后直接遍历一次原始数据字节得到data，完成后再将length写到data之前空出来的字节处。

基于这一前提，他们得出这样的算法，能够以大概率做到只需遍历一次原始数据。

```
Get the length of an input string (lo) and its byte count (bo).
∙ Prepare a buffer whose length is lo + bo.
∙ Guess the result length with the factor of 0.8 (le) and its byte count (be).
∙ Huffman encode the string starting from the position of be.
– If we reach the end of the buffer during this encoding, encode lo in the beginning of the buffer and copy the input string as is.
– Otherwise, obtain the real length of the result (lr) and its byte count (br).
    * In the rare case where be is not equal to br, move the result appropriately within the buffer and encode lr in the beginning of the buffer.
    * Otherwise, encode lr in the beginning of the buffe

```

## Huffman decoder

关于Huffman decoder的实现，工业界主流实现无一例外都是基于2003的年这篇论文[Fast Prefix Code Processing](https://www.semanticscholar.org/paper/Fast-prefix-code-processing-Pajarola/004a8f357a8f6f5ee16d355d7923ac0e817b0a96)中提出的Fast prefix code decoding算法（姑且简称为FPCD），下面先介绍下这个聪明并且有效的算法。

我们知道huffman decoding最naive的方法是对编码后的数据按照每个bit来访问huffman树，如果访问到叶子节点，说明成功解码出一个字符并输出，否则按照下一个bit访问二叉树上当前节点的左右子节点。这样做的问题在于只能按照一次一个bit来解码，显然不太能够利用现代硬件按照word处理数据的计算能力。

FPCD算法目的是要做到每次能够对若干个bits进行同时处理，其算法的提出是基于这样的观察：对于一个huffman二叉树而言，树上的每个节点都是一个状态，我们可以根据一颗给定的huffman树预先计算出从某个状态出发，接收k个bits的所有可能的输入后所处的状态，以及其产生的decoding输出。举个简单的例子，比如现在有这样一个huffman树：

![huffman tree](/public/fig/huffman.png)

如果我们每次处理两个bits的话，可以得出如下图所示的一个状态迁移图：

![huffman fsm](/public/fig/huffman_fsm.png)

假设目前处于状态0, 接收的输入数据是00，会使得状态迁移至状态3 (经过1), 并且成功解码输出A字符；如果接收的输入11，会使得先经过状态2解码出C字符，然后停在状态1，以此类推，可以得到所有状态针对n bits 所有取值的状态迁移表。

可以看出所有叶子节点的状态其实和root节点是一致的，等价于状态0，因此对于alphabet大小为N的huffman码表，由于huffman树非叶子节点个数是N-1（huffman树是正则二叉树: N_0 = N_2 + 1）, 所以状态迁移表中也就是有N-1个状态，每个状态有2^k个状态转移的可能，所以整个状态迁移表的大小是 (N-1)* 2^k。给定一颗huffman树，我们就可以预先计算出这个状态转移表来，表中的每个entry由三部分组成：
- next state
- 一些标志位，用于决定是否有输出，以及是否解码完成，是否解码失败
- 要输出的bytes



有了这个预先计算好的FSM，decode算法过程就是：
```
- 从root state开始，依次处理每k个bits
- 根据当前state以及当前k个bits的取值，在状态迁移表中找到对应的entry，跳转到entry的next state，并根据entry的标志位决定如下动作：
    - 如果标志位中有输出标志，则输出entry中对应的byte
    - 如果标志中有解码失败标志，则终止解码过程，直接通知上层解码失败
- 处理最后4个bits后，如果entry中没有解码完成标志，说明数据有丢失，通知上层解码失败
```

netty中使用k是4，即一次性处理4个bit输入，选择4是空间和时间的trade off，另外对于http2的静态huffman码表来锁还有个好处是每次状态转移至多输出一个byte（因为http2的huffman码表中最短的code也有5bit）。具体decode代码如下：

```
    //处理单个byte的输入
    public boolean process(byte input) {
        return processNibble(input >> 4) && processNibble(input);
    }

    private boolean processNibble(int input) {
        // The high nibble of the flags byte of each row is always zero
        // (low nibble after shifting row by 12), since there are only 3 flag bits
        int index = state >> 12 | (input & 0x0F);
        state = HUFFS[index];
        if ((state & HUFFMAN_FAIL_SHIFT) != 0) {
            return false;
        }
        if ((state & HUFFMAN_EMIT_SYMBOL_SHIFT) != 0) {
            // state is always positive so can cast without mask here
            dest[k++] = (byte) state;
        }
        return true;
    }
```

需要说明的是netty里状态迁移码表中的entry是由前面说的三个元素pack而成的32bit integer。
最高16位是next state，然后8位是flags，最后8位是输出byte。因此计算迁移表index的时候需要先将当前状态变量state 右移16，得到unpack后的实际state，所以index 需要通过(state >> 16 * 16 + input & 0x0F)得到，用位运算简化下就是 index = state >> 12 | (input & 0x0F)。


## 总结
以上就是HPACK中static huffman codec的工业界实现，想实现一个高效的huffman codec需要一些trick