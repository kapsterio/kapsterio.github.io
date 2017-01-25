---
layout: post
title: "编译原理课堂笔记"
description: ""
category: [lesson]
tags: [compiler]
---
{% include JB/setup %}


## 自顶向下分析：
从开始符号出发推出句子，判断句子是否和给定的目标句子相等，过程中会涉及回溯。可以通过前看符号来避免回溯


算法伪码：

{% highlight c linenos %}
tokens[];
i=0;
stack = [s] //s是开始符号
while (stack !=  []) {
    if (stack[top] is a terminal t) { //如果是终结符
        if (t == tokens[i++]) 
            pop();
        else backtrack(); //t不能匹配tokens输入流的当前符号，因此回溯，回溯涉及将产生包含t的产生式的所有右部出栈、左部压栈，游标前移
    } else if (stack[top] is a nonterminal T) { //非终结符
        pop();
        push(the next right hand side of T); //将右部逆序压栈，这里如果没有下一个right hand side了怎么办————回溯
        if (there is no rhs of T any more) backtrack(); //这里回溯掉产生包含T的的产生式。可能需要另外一个栈来记住产生式选择以方便回溯
    }
}
{% endhighlight %}


算法比较昂贵，我们需要线性时间的算法：

- 避免回溯
- 引出递归下降分析算法和LL1分析算法

## 递归下降分析：
- 每个非终结符对应一个分析函数。分析函数的结果是当前输入token能否被代表这个函数的产生式接受（吃掉）。在每个分析函数内部用当前的前看符号选择一个产生式，如果没得选择报错。接下来，对这个选择的产生式进行分析，假设产生式中即包含终结符也包括非终结符，那么对于终结符判断当前输入符号是否匹配，对于非终结符递归调用该非终结符的parse函数。

- 问题：怎么根据前看字符来选择产生式？
对算术表达式的递归下降分析

{% highlight c linenos %}
E -> E + T
    | T
T -> T * F
    | F
F -> num
{% endhighlight %}


可以看成： E -> T + T + T + .... + T, T -> F * F * ... * F

{% highlight c linenos %}
parse_E() {
    parse_T();
    token = tokens[i++];
    while (token == '+') {
        parse_T();
        token = tokens[i++];
    }
}
{% endhighlight %}

{% highlight c linenos %}
parse_T() {
    parse_F();
    token = tokens[i++];
    while (token == '*') {
        parse_F();
        token = tokens[i++];
    }
}
{% endhighlight %}

- 为什么递归下降只需扫描一次输入流，看书时思考下
递归下降其实就是上一节的自顶向下的算法的递归版本，区别在于对非终结符产生式的选择，自顶向下算法中栈模拟的就是这个递归调用过程，产生式的选择也是盲目的一个个尝试。

## LL(1)分析算法
从语法声明式规范自动生成语法分析器代码的工具：ANTLR、YACC、bison

### 表驱动的分析算法
行：非终结符
列：终结符
表中元素Xij代表：栈顶元素为i行时，遇到j列非终结符时，选择产生式Xij

### 表怎么生成
定义非终结符的first集:

- 对 N -> a... : first(N) U= {a}
- 对 N -> M... : first(N) U= first(M) 

伪码：(不动点迭代算法)

{% highlight c linenos %}
foreach (nonterminal N) {
    first (N) = {};
}
while (some set is changing) { //迭代的终止条件是没有集合发生变化
    foreach (production p: N -> beta1 .. betan) {
        if (beta1 == a) {
            first(N) U= {a};
        }
        if (beta1 == M) {
            first(N) U= first(M);
        }
    }
}
{% endhighlight %}

思考：怎么证明算法正确且能终止

定义任意串的first集：

{% highlight c linenos %}
first_s(beta1...betan) = 
    first(N), if beta1 == N;
    {a}, if beta1 == a;

{% endhighlight %}

构造LL(1)分析表：
遍历每个production p，将分析表中行位p左部，列在p右部的first集中的元素填上p

定义NULLABLE集：
非终结符X属于集合NULLABLE，当且仅当：

- 基本情况：
    X -> 
- 归纳情况：
    X -> Y1 ... Yn, Y1 ... Yn都属于NULLABLE集

NULLABLE集构造算法：

{% highlight c linenos %}
NULLABLE = {};
while (nullale is still changing) {
    foreach (production p : X -> beta)
        if (beta == '')
            NULLABLE U= {X};
        if (beta == Y1 ... Yn) 
            if (Y1 belong to NULLABLE && ... && Yn belong to NULLABLE)
                NULLABLE U= {X};
}

{% endhighlight %}

现在重新redine下非终结符的first集的定义：

- 基本情况:X -> a 
    firsr(X) U= {a}
- 归纳情况:X -> Y1 ... Yn
    - first(X) U= first(Y1)
    - if (Y1 belong to NULLABLE), first(X) U= first(Y2)
    - if (Y1, Y2 belong to NULLABLE), first(X) U= first(Y3)
    - ....

那么非终结符first集构造算法redine后如下：
{% highlight c linenos %}
foreach (nonterminal N)
    first(N) = {};
while (some set is changing) {
    foreach (production p : N -> beta1 ... betaN)
        foreach (betai from beta1 to betan) { //相比原来多了一层对产生式右部的符号的迭代
            if (betai == a...) {first(N) U= {a}; break;}
            if (betai == M...) {
                first(N) U= first(M);
                if (M is not in NULLABLE) break;
            }
        }
}

{% endhighlight %}

思考：怎么证明算法正确，并且能终止？1）集合元素有限，因此算法肯定能终止 2）反证法证明当算法终止时，结果集合就是要求的目标集合




OK, 有了非终结符的first集完整定义和构造算法后，来分析串的first_s集：

原来的定义是：对于N -> beta1 ... betan
first_s(beta1...betan) = 
    first(M), if beta1 == M;
    {a}, if beta1 == a;

现在beta1可能是NULLABLE，同样beta2、...、betan都可能是NULLABLE的，first_s除了可能包含betai的first集外，还可能包含串之外的非终结符，即follow(N)。因此引入follow集的概念

非终结符follow集：非终结符后可能跟随哪些终结符构成的集合

这里可以先定义每个产生式右部符号的temp集,对于产生式 N -> beta1 ... betan而言：

基本情况: temp(betan) = follow(N)

归纳情况: 

- if betai+1 is terminal, temp(betai) = {betai+1}
- if betai+1 is nonterminal && betai+1 is not NULLABLE, temp(betai) = first(betai+1)
- if betai+1 is nontermianl && betai+1 is NULLABLE, temp(betai) = first(betai+1) U temp(betai+1)

temp集是干嘛的呢？每个非终结符的follow集必定包含它所在的每个产生式中它的temp集。
因此，对于非终结符的follow集构造算法如下：

{% highlight c linenos %}

foreach (nonterminal N) 
    follow(N) = {};
while (some set is changing) {
    foreach (prodcution p : N -> beta1 ... betan) {
        temp = follow(N); //定义temp集，每轮循环开始前temp是可能跟在betai后的非终结符集合
        foreach (betai from betan downto beta1) {
            if (betai == a...) temp = {a};
            if (betai == M...) {
                follow(M) U= temp;  //M的跟随集里增加新元素
                if (M is not NULLBALE)
                    temp = first(M); //M不在NULLABLE内，因此betai-1的temp集只能是first(M)
                else temp U= first(M); //M在NULLABLE内，betai-1的temp集是first(M) U temp(betai)
            }
        }
    }

}

{% endhighlight %}

最后，给出串的first_s集算法：

{% highlight c linenos %}
foreach (production p) 
    first_s(p) = {};

calculate_first_s (production p : N -> beta1 ... betan) {
    foreach (betai from beta1 to betan) {
        if (betai == a ...) { 
            first_s(p) U= {a};
            return;
        }
        if (betai == M ...) {
            first_s(p) U= first(M);
            if (M is not NULLABLE) return;
        }
    }
    first_s(p) U= follow(N);//为什么需要并上follow(N)，此时p可以推出空，结合下first_s的作用就知道了为什么了
}

{% endhighlight %}


现在，终于可以构造LL(1)分析表了，其实和之前完全一样。那么LL(1)分析器的算法框架如下：

{% highlight c linenos %}

tokens[];
i=0;
stack = [s] //s是开始符号
while (stack !=  []) {
    if (stack[top] is a terminal t) { //如果是终结符
        if (t == tokens[i++]) 
            pop();
        else error(..) //如果不匹配则直接报错
    } else if (stack[top] is a nonterminal T) { //非终结符
        pop();
        push(table[T, tokens[i]]); //将table[T, tokens[i]]号产生式的右部逆序压栈
    }
}

{% endhighlight %}

### 冲突处理
改变文法以去掉冲突
- 消除左递归（一般变成右递归）
- 提取左公因子


### LL（1）分析算法的缺点
- 文法类型受限
- 可能需要文法改写

## LR分析算法 （移进-归约）
自底向上
YACC、bison、CUP
最右推导的逆过程