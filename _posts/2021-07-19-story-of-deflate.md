---
layout: post
title: "story of Deflate"
description: ""
category: 
tags: []
---

## Deflate算法以及ZIP、GZIP、ZLIB

在无损压缩算法中Deflate基本上互联网时代无损压缩算法的事实标准，我们日常生活常用的zip、gzip等工具、http协议中对body部分的常见压缩算法都是使用的deflate。Deflate算法最早是由[Phil Katz](https://zh.wikipedia.org/wiki/%E8%8F%B2%E5%B0%94%C2%B7%E5%8D%A1%E8%8C%A8)在编写PKZIP(也就是后来被广泛使用的ZIP)时提出，后来在众多自由软件先驱的努力下被众多广泛实现和应用，再被IETF标准化，形成[Deflate压缩标准](https://datatracker.ietf.org/doc/html/rfc1951)。Deflate算法本身也是在前人工作的基础上创新得到，主要基于[LZ77算法](https://en.wikipedia.org/wiki/LZ77_and_LZ78)和大名鼎鼎的[Huffman coding算法](https://en.wikipedia.org/wiki/Huffman_coding)，Zlib的作者在[An Explanation of the Deflate Algorithm](https://zlib.net/feldspar.html)里对Deflate的工作原理做了很好的解释。接下来的主要内容也将会是结合Deflate原理，整理、学习和分析Deflate的一些design choices。不过进入正题之前，先来理清下这些常见名词到底是什么，我觉得应该有很多人和我一样被中文互联网上deflate、zlib、zip、gzip、tar的这些名词的混用而搞的晕乎乎的。


### Deflate
Deflate前面提到了，是源自于DOS上PKZIP这个工具的无损压缩算法，后来在进入互联网时代后被IETF标准化了。我们知道算法和数据结构是密不可分的，IETF对Deflate的标准化不仅将其压缩策略标准化了（比如怎么LZ77以及huffman coding、选择什么样的huffman树），同时还对压缩数据的格式做了标准化。因此Deflate在当下的语义下不仅代表一种压缩算法、还是一种压缩文件的数据格式。如今Deflate标准已经是众多压缩软件/工具的基石。


### ZIP
[ZIP](https://zh.wikipedia.org/wiki/ZIP%E6%A0%BC%E5%BC%8F)就是应用Deflate最早的压缩工具，前面提到ZIP是由传奇程序员Phil Katz于1989年relase在DOS平台，名叫PKZIP。在1989年这个让人忍不住想念上两句诗的年份，Internet还处于雏形阶段，WWW尚未诞生，PKZIP在当年作为一个压缩DOS系统下的文件夹的压缩工具，它也定义了一种文件归档的格式，经由它压缩的文件后缀是.ZIP。在其内部应用其首创的Deflate算法对文件夹内每个文件进行压缩，最后再文件中维护一个目录结构，然后再附带个CRC-32的checkcode。由于每个文件独立压缩，用ZIP压缩的文件夹能够随机读取每个文件进行解压，另一方面他最终产生的压缩文件体积也相对较大。在windows进入图形时代后，Winzip一开始作为pkzip的ui前端逐渐流行开来，后来被内嵌到windows系统，成为绝大部分windows用户的默认压缩解压工具。PKZIP并非开源软件，使用的话是需要商业license授权的，后来Phil Katz为了ZIP格式的互操作性，公开了.ZIP文件格式的技术标准, .ZIP格式在整个ZIP生态的发展下广泛流行起来。与此同时自由软件先驱们发起了[Info-ZIP计划](https://zh.wikipedia.org/wiki/Info-ZIP)，其包含多个Zip、UnZip等多个命令的开源软件包，目前大多数linux/unix系统中内置的zip/unzip命令都是Info-ZIP中的Zip和Unzip。


### GZIP
回到Unix这边，Unix哲学是一个命令就干一件事，所以在正统Unix观点看来，压缩整个文件夹其实是两件事。第一打包整个文件夹形成一个归档文件，第二压缩这个归档文件。因此有了[tar](https://en.wikipedia.org/wiki/Tar_(computing))和[compress](https://en.wikipedia.org/wiki/Compress)。compress由于压缩基于的被专利保护的[LZW算法](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch)，一直存在专利的争议。gzip诞生的目的就是用来替代unix里的compress，目前已经是unix/linux系统下压缩的事实标准。Gzip中的G很明显stand for GNU, it's free! 它由Jean-loup Gailly和Mark Adler于1992年开发，后其定义的文件格式也被IETF标准化在[RFC 1952](https://datatracker.ietf.org/doc/html/rfc1952)。从实现上来说，Gzip主要基于deflate算法，也能被很扩展使用其他算法，因此不涉及到任何专利（虽然Phil Katz对PKZIP中deflate实现申请了专利，但是由于其公开了ZIP的数据格式标准，因此是存在编写一个不侵犯任何专利的Deflate实现的可能的）。和ZIP不同，Gzip不是一种文件夹归档的工具/格式，它不能直接用来压缩文件夹的多个文件，需要和tar这种归档工具结合使用，这也就是为了什么linux/unix下最常见的压缩文件后缀一般是.tar.gz。由于压缩的是整个归档后的文件夹，因此在解压的时候它不能想ZIP那样随机访问某个文件进行解压，但是它产生的最终压缩文件大小也会相对小些。

### ZLIB
GZIP的作者后来将其deflate的实现抽离开来，形成一个lib以便供其他文件格式使用，其中最常见使用这个压缩库的就是PNG图片格式。这个lib就是大名鼎鼎的zlib，目前被广泛应用于各种程序中，大部分http server使用zlib进行body数据的压缩和解压。zlib库本身并没有和gzip绑定，zlib库本身也提供了一种轻量级的zlib wrapper数据格式，后被IETF标准化在[RFC 1950](https://www.ietf.org/rfc/rfc1950.txt)中。那么zlib这种格式轻量在那呢，作者在zlib官网的faq中给出了[回答](https://zlib.net/zlib_faq.html#faq19)，具体想了解则需要去读下RFC 1950和RFC 1952标准了。


### 总结
OK，总结一下：Deflate是一种基于LZ77和霍夫曼编码的通用数据压缩算法，诞生于PKZIP，后被[RFC 1951](https://datatracker.ietf.org/doc/html/rfc1951)中标准化，目前Deflate通常指其压缩算法和其压缩数据格式，姑且称之为原生delfate格式(raw deflate data format)。ZIP目前指一种包括文件夹压缩和归档数据格式，多种各个操作系统平台上都有很多工具支持读写ZIP格式归档文件的工具（例如windows下WinZip, Unix/linux下的zip/unzip等等）。GZIP是针对单个文件的通用数据压缩工具，其实现了Deflate算法，此外它还定义了gzip压缩文件的标准格式，在[RFC 1952](https://datatracker.ietf.org/doc/html/rfc1952)中被标准化，其是对raw deflate data format的一种封装，姑且称之为gzip data format。Zlib一开始是源自于GZIP 工具的deflate算法实现库，后来它也定义了一个更轻量级的zlib的压缩文件数据格式，也是一种对raw deflate data format的轻量级封装，在[RFC 1950](https://www.ietf.org/rfc/rfc1950.txt)中被标准化了，姑且成为zlib data format。

OK，最容易产生迷糊的地方来了，就是在HTTP1.1协议里面的[content coding](https://datatracker.ietf.org/doc/html/rfc2616#section-3.5)这节，这里定义了http1.1协议需要支持的几种内容压缩数据格式。但是在命名上非常容易让人产生误解：

>    gzip:
>
>    An encoding format produced by the file compression program "gzip" (GNU zip) as described in RFC 1952 [25]. This format is a Lempel-Ziv coding (LZ77) with a 32 bit CRC.
>
>   compress:
>
>   The encoding format produced by the common UNIX file compression program "compress". This format is an adaptive Lempel-Ziv-Welch coding (LZW).
> 
>   Use of program names for the identification of encoding formats is not desirable and is discouraged for future encodings. Their use here is representative of historical practice, not good design. For compatibility with previous implementations of HTTP, applications SHOULD consider "x-gzip" and "x-compress" to be equivalent to "gzip" and "compress" respectively.
>
>  deflate:
>
>  The "zlib" format defined in RFC 1950 [31] in combination with
>  the "deflate" compression mechanism described in RFC 1951 [29].
>
>  identity:
>
>  The default (identity) encoding; the use of no transformation whatsoever. This content-coding is used only in the Accept-Encoding header, and SHOULD NOT be used in the Content-Encoding header.

看到没，http1.1定义中“gzip”指的是RFC 1952中的gzip data format，而"deflate"则指是RFC 1950中的zlib data format。问题就出在这，有些粗心的http1.1实现将其误认为是RFC 1951中定义的raw deflate data format，其中最臭名昭著的就是田厂。因此即便是"deflate"相比于"gzip"更加轻量化和高效，大部分client和server选择"gzip"作为内容压缩格式，至少所有浏览器都能正确对待它。在zlib官网的faq中作者解释了[为什么"gzip"在http1.1里用的更多](https://zlib.net/zlib_faq.html#faq39)



