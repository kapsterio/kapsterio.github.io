---
layout: post
title: "如何按照给定qps模拟user-generated请求"
description: ""
category: 
tags: [Poisson distribution, Exponential distribution]
---
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default" async></script>

## 背景
写这篇博客的起因是我最近对微博消息箱长连推送服务进行了整体架构层面的重构，然后需要对新推送系统的消息推送能力进行压测。压测需要通过控制几个变量来对推送系统造成不同的负载压力，得出系统在不同负载水平下消息推送的各分位推送耗时、成功率等几个指标，这里主要的变量有：
- 长连接数
- 消息的字节大小
- 消息的扇出连接数(fanout)，控制每条消息需要推给几个长连接
- 写入推送系统的消息qps (write qps)，最终推送系统的推给客户端长连接消息qps是： write qps * fanout

这其中很重要一步是压测系统要去按照给定的write qps去生成消息流写给推送系统，怎么实现这步，最朴素的做法是通过给定的qps，计算一个固定的消息时间间隔，然后按照这个时间间隔生成消息流即可。这种做法显然能满足要求，但是不够真实，对系统造成的压力太regular。同样qps，系统在这种流量负载下的表现情况可能会和真实流量情况下表现不一致，因此我们希望能尽量模拟给定qps下的真实流量分布，随机变量的概率分布这时就派上用场了。

<!--more-->

## 模拟生成给定qps的流量实现
实现上我参考了grpc-java的benchmarks里[压测客户端](https://github.com/grpc/grpc-java/blob/master/benchmarks/src/main/java/io/grpc/benchmarks/qps/OpenLoopClient.java)，这里先给出代码实现:

```java

 private static final double DELAY_EPSILON = Math.nextUp(0d);

    /**
     * 模拟服从参数为targetQps指数分布的时间间隔
     * @param targetQps
     * @return
     */
    private static long nextDelay(int targetQps) {
        double seconds = -Math.log(Math.max(random.nextDouble(), DELAY_EPSILON)) / targetQps;
        double nanos = seconds * 1000 * 1000 * 1000;
        return Math.round(nanos);
    }

    /**
     * 按照给定的qps持续产生消息
     */
    private void simulateMessageDistribute(int duration) {
        long now = System.nanoTime();
        long nextMessage = now;

        long i = 0;
        while (i < duration * qps) {
            now = System.nanoTime();
            if (nextMessage - now <= 0) {
                nextMessage += nextDelay(qps);
                executorService.submit(this::generateMessage);
                i ++;
            }
        }
    }
```
可以看出这段代码要做的事很简单，simulateMessageDistribute方法就是按照给定qps持续产生消息，持续时间由duration给定。实现上其通过nextDelay模拟一个随机的两次消息之间的时间间隔，每次生成一条消息后，cpu轮询至这个间隔时间达到后再生成新信息。这段代码的核心就在于怎么去模拟出给定消息qps下两条消息的时间间隔。熟悉指数分布的同学应该能立刻明白这个时间间隔的取值就是一个服从一个参数为qps的指数分布，下面就来粗浅的聊聊概率分布相关的知识。

## 随机变量的概率分布
学过大学本科概率论与数理统计的同学都知道，研究随机变量的概率分布是概率论中最为基础内容。随机变量分为两类，离散型随机变量和连续型随机变量，常见的离散型随机变量概率分布有：

### Bernoulli distribution 

中文叫伯努利分布，又叫两点分布、0-1分布。其随机变量取值是一次伯努利试验的结果，即取值要么1，要么是0, 如果取值为1的概率p，那么0的概率就是 1-p。

其概率质量函数是:

$$f(x;p) = p^x (1-p)^{(1-x)}$$

其期望是：

$$E[X] = \sum_{i=0}^{1} x_i f(x) = 1 * p + 0 * (1-p) = p$$

其方差是：

$$var[X] = \sum_{i=0}^{1}(x_i - E[X])^2 f(x) = (0-p)^2 (1-p) + (1-p)^2 p = p(1-p) = pq$$

### Categorical distribution

Categorical distribution是Bernoulli distribution在多个类别情况下的推广。当随机变量有k个取值，{1,2,...,k}，概率分别为\\(p_1,p_2,...,p_k\\)，其中 \\(\sum_{i=1}^{k}p_i = 1\\) 

概率质量函数可以表达为：

$$f(x;p_1,...,p_k) = \prod_{i=1}^{k} p_{i}^{[x=i]}$$

其中

$$\begin{equation}
[x=i]=\left\{
\begin{aligned}
1 &  & x=i \\
0 &  & 其他 \\
\end{aligned}
\right.
\end{equation}$$

将概率质量函数写成这种统一形式对很多计算过程（如求似然函数）都有帮助


### Binomial distribution
二项分布的随机变量取值是多次独立重复伯努利试验的结果，相比于伯努利分布，它的参数多了个n，即伯努利试验重复的次数。

其概率质量函数是：

$$f(k;n,p) = Pr(X=k) = {n\choose k}p^k(1-p)^{n-k}$$

其中:

$${n\choose k}=\frac{n!}{k!(n-k)!}$$

其期望是：$$E[X] = np$$, 方差：$$Var[X] = np(1-p)$$

即它的期望值和方差分别等于每次单独试验的期望值和方差的和


### Multinomial distribution

多项分布，二项分布在多个类别情况下的推广


### Geometric distribution

随机变量是获得一次成功结果需要进行的伯努利试验的次数， 

概率质量函数为：

$$P(X=k) = q^{k-1}p$$

期望是: $$1/p$$,  方差是$$q/p^2$$


### Possion distribution

[泊松分布](https://en.wikipedia.org/wiki/Poisson_distribution)，随机变量是在一个固定时间间隔里某个事件独立发生的次数， 是二项分布的极限n -> 无穷， np -> λ， λ是这个固定时间间隔内事件发生的期望次数或者平均次数

- Poisson point process or  Poisson process
    - 随机事件发生的过程的建模
    - counting process (将Poisson过程解释成一个在一段时间内对随机事件发生次数计数过程，在每一时刻t，N(t)表示随机事件在0-t这段时间内发生的次数, N(t)服从Poi(λt)，也就是说Poisson过程在每个时刻的取值都服从一个Poission分布, 也就是说如果将N(t)当做随机变量的话，E(N(t)) = λt）
    - point process (将Poisson过程定义在一个一维区间(a,b]上，考虑在这个区间上的随机点数，N(a,b)~ Poi(λ(b-a)), 和计数过程很类似）

### Exponential distribution 
指数分布的随机变量是Poisson过程中两次事件发生的间隔，这个时间间隔服从指数分布，参数和poisson分布一样，也是λ, 

概率密度函数为：

$$\begin{equation}
f(x;λ)=\left\{
\begin{aligned}
λe^{-λx} &  & x>=0 \\
0 &  & x<0 \\
\end{aligned}
\right.
\end{equation}$$


概率累积分布函数为(CDF):

$$\begin{equation}
F(x;λ)=\left\{
\begin{aligned}
1- e^{-λx} &  & x>=0 \\
0 &  & x<0 \\
\end{aligned}
\right.
\end{equation}$$


期望是 \\(1/λ\\)，方差是 $$1/λ^2$$，λ是单位时间间隔内事件发生的次数


## user-generated traffic生成
有了这些背景知识，考虑下面这个问题：如果模拟给定的qps的user-generated traffic？

根据Poisson过程的定义，user-generated reqeust事件的发生过程就是一个Poisson过程，即在一段时间内请求次数服从一个Poisson分布，更具体点，qps为3000，那么在一秒内实际产生的请求数肯定会在3000附近波动，这个数服从参数λ为3000的泊松分布。按照参数λ的指数分布来随机产生事件间隔序列，每个间隔产生一次request，这个request产生过程就是Poisson过程。λ是什么？就是qps。那么问题就在于怎么生成一个服从指数分布的随机时间间隔序列？也就是怎么产生一个服从指数分布的随机变量。这是一个通用问题：即给定按照随机变量的概率分布产生随机数，有很多种算法，其中一种很常用的方法叫做 [inversion method](http://www.mathematik.uni-ulm.de/stochastik/lehre/ss06/markov/skript_engl/node29.html)

### inversion method
通过一个均匀分布的随机变量U、以及一个函数映射$$Fx$$, 来产生另外一个随机变量X，这个新的随机变量X服从我们需要模拟的概率分布F, F是X的CDF，其中:
$$Fx = F^{-1}$$

下面来证明X服从CDF为F的概率分布：

证明如下：

$$P(X <= x) = P(Fx(U) <= x) = P(F^{-1}(U) <= x) = P(U <= F(x)) = F(x)$$

证毕

倒数第二步利用的是CDF函数单调递增的特性，最后一步是利用0-1区间均匀分布的特性，都很直观

上文中的那段代码nextDelay函数就是生成服从参数为qps的指数分布的随机变量。




