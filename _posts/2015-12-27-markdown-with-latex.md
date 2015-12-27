---
layout: post
title: "markdown with latex"
description: ""
category: [prductivity]
tags: [markdown, latex]
---
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default" async></script>

## markdown是个利器
自从接触了markdown，就很少再用Microsoft word或者apple pages写文档了. markdown能让你更关注内容，至于格式排版什么的就交给markdown去操心吧（和Latex理念一样）。这很重要，因为在写文档时这让能你消除对鼠标的依赖，从而提高效率，毕竟拖着鼠标来回点很费事的好么~ 至于编辑器，用vim就足够了。

我在平常工作中主要使用markdown来写邮件和博客，也就是说只涉及把markdown文件解析为html文件，有一打的markdown解析引擎可以干这个事。

<!--more-->

然而markdown也不是万能的，比如想在文档中插入漂亮的数学公式就比较费事。回想起研究生时用latex来写公式的体验还是蛮不错的，语法也比较简单，而且还能从wikipedia导入（偷）一些现成的。那么能不能在markdown里能像在latex里一样直接书写公式，而不用去操心公式的排版呢，搜索了下，已有轮子了: [pandoc](http://pandoc.org/)。 研究了下，发现这货简直就是瑞士军刀啊！ 不仅支持markdown<->html、markdown<->tex，最重要的还支持markdown->pdf！

## pandoc
> If you need to convert files from one markup format into another, pandoc is your swiss-army knife.
——John MacFarlane, author of pandoc

### 安装
Mac下可以使用homebrew安装，不嫌费事的话也可以从源码安装。另外，pandoc还为mac提供了[pkg安装包](https://github.com/jgm/pandoc/releases/download/1.15.2/pandoc-1.15.2-osx.pkg)。

装好后打开终端输入`pandoc --version`看下版本。

### 使用
终端输入：

```>
	pandoc example.md -f markdown -t html -o example.html -s 
```

将example.md装换成example.html文件。也可以反过来将html装换为md文件

终端输入：

```>
	pandoc example.md -f markdown -t tex -o example.tex -s 
```

则可以将md文件装换为tex文件，反过来也是可以的。

### 生成pdf
如果想装换为pdf呢，如果你装了latex（比如Mac平台的MacTeX，windows平台的MiKTeX)，那么就可以将从tex文件生成pdf，不过这样也挺费事的。好在pandoc提供了直接从md到pdf的功能：

```>
	pandoc example.md -f markdown -t pdf -o example.pdf -s 
```

bingo~ pdf文件生成了。

不过这前提是你得装了MacTeX，但是完整的MacTeX太大，2个多G，包括三个部分：

- 完整的TeX Live 2015 （大小2个G）
- GUI程序：前端、命令集等等
- Ghostscript 9.16

还好有个精简版的tex：[BasicTeX](http://www.tug.org/mactex/morepackages.html)，它拥有所有MacTeX的命令集、一个100M的Tex Live子集，对于我们来说是够了，下载pkg安装包后会安装到 `/usr/local/texlive/2015basic`目录下，装好了后记得把`/usr/local/texlive/2015basic/bin`加到PATH环境变量里，这样pandoc也就能找着pdflatex命令了。

[这是那个example.md](http://mplewis.com/files/pandoc-md-latex/example.md)

[这是生成的example.pdf](http://127.0.0.1:4000/example.pdf)

### 网页上支持公式
现在我想在我这个git pages博客里支持矢量公式，然而git pages的博客是通过jekyll解析博客内容的markdown文件生成的，咋办？大家看过[Stackoverflow](http://stackoverflow.com/)上的公式吧。这就用到MathJax引擎了。在需要显示矢量公式的博客markdown文件上加上：

{% highlight javascript linenos %}
 <script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>
{% endhighlight %}

然后再使用TeX语法写公式， `$$equation$$`表示行间公式， `\\(equation\\)`表示行内公式。

比如行间公式：

{% highlight c linenos %}
$$x=\frac{-b\pm\sqrt{b^2-4ac}}{2a}$$
{% endhighlight %}

显示的结果是：

$$x=\frac{-b\pm\sqrt{b^2-4ac}}{2a}$$

行内公式： 

{% highlight c linenos %}
\\(-b\pm\sqrt{b^2-4ac}\\)
{% endhighlight %}

显示的结果是： \\(-b\pm\sqrt{b^2-4ac}\\)

<br/>

**世界因为有这些工具变得更美好了些~**

