---
layout: post
title: "Basic of Elasticsearch"
description: ""
category: 
tags: [lucene,elasticsearch,full text search]
---
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default" async></script>

# Introduction

## es是什么

Elasticsearch是基于Apache lucene的分布式的开源检索引擎，lucene本身只是全文检索的java lib，直接拿来用的话需要集成到自己的应用当中，而es则是一个完整的系统，类似的还有apache的solr。es的索引和搜索部分是完全基于lucene实现，此外它在lucene之上主要提供了如下几个方面的功能：

 1） 分布式：es集群可以很方便的横向扩展以及做到高可用，这在大数据时代是必备的。es的横向扩展能力包括两个方面，索引文件的存储分布在集群的多个节点上，以及搜索处理在多个节点间负载均衡。高可用也是分布式系统必须考虑的问题，和其他系统一样，es通过副本来解决某个节点down掉后索引不可用问题。

<!--more-->

 2） 基于lucene，提供更为简单易用、功能多样的实时搜索api （query）

 3） 提供了实时分析、聚合的功能 (aggregation)

本文主要关注两个方面：indexing和searching

## 简单谈谈es的分布式（shard & replica）
- es把索引文件划分成不同的shards，shards分布于机器中一个节点或多个节点之上
- es可以配置shards的副本数，即每个shard都可以多个副本，以防止节点failure导致的索引数据丢失，副本也是分布于集群内各个节点上
- indexing和searching（写入和检索）会在各个shard间负载均衡。对于indexing（写请求），请求会被转发到对应shard主副本所在的节点(master)，然后再同步到各个其他副本(follower)。对于searching（读请求），任何副本都可以接受请求，一个shard的多个副本间可以通过简单的round-robin方式来进行负载均衡
- 集群中的任意节点都可以用来接受index或者search的请求，接受请求的节点扮演协调节点，由协调节点处理分析、向各个执行节点（当然也可能包括自己）发起请求、汇总数据、返回结果给客户端
- 当新节点加入集群或者集群中有节点失败时的平滑索引数据迁移（shards redistributing）

再继续介绍ES的indexing和searching之前，有必要先介绍下es的文档模型，尤其注意的是它和关系型数据库数据模型的异同。

## 基于lucene的文档模型

### 逻辑结构
如果说关系型数据库的逻辑存储单元是一条条记录(record)的话，对应到es里的逻辑存储单元就是一个个文档(document)。document是lucene在indexing和searching时的逻辑单元，一个document可能包含一个或者多个fields，每个field的内容是我们的真正存储的数据（比如一篇文章有标题、内容、作者、日期等等fields），可能是整形、浮点型、字符串等等，另外可以针对field设置一系列选项用于告诉lucene怎么处理一个doc的这个field的内容（包括是否存储、是否需要被检索、是否分词、怎么分词等等）。lucene的倒排索引也是以field为单元的，在searching时自然也是以field为单元来进行检索的。

与关系型数据库不同的是，lucene没有一个严格的schema的概念，意味着document的结构是很松散的，两个document中的field可以有很大差别

ES中引入了index和type的概念，分别对应着关系型数据库中的库和表，使得更易管理索引，也更符合关系型数据库的使用习惯。另外ES中还引入了文档id的概念，使得我们可以手动指定doc的唯一标识或者让es自动为每个doc生成一个id，从而让我们可以通过id来实时地对文档进行CURD。

### 倒排索引
我们知道关系型数据库里的索引实现时采用的数据结构一般是B-树，而全文检索领域索引用的最多的数据结构则是倒排索引，倒排索引字面上意思就已经表示了它是一个由term反向映射到document这样的一种索引。它由两部分组成：dictionary（字典）和postings（倒排表）。

dictionary很简单就是所有的term构成的集合，postings则是每个term对应的document list，最简单的postings的document list中只有doc id有序列表，用于解决boolean query问题（不涉及排序，仅回答term在哪些doc中出现）。实际应用中postings文件内容还可以包括term在doc中出现的频率、位置等等其他信息，用于解决更复杂的query、排序问题

有了倒排索引后，对于boolean query问题就变成了一个简单的有序list求交、求并问题，最简单的实现就是merge sort中的merge过程。lucene中有着各种各样的fliter来解决各种boolean query问题，filter则是基于bitset来实现纯内存的高效的文档集合运算，性能非常之高（也正是因此，lucene里的一条金科玉律就是能用filter的地方就不要用query，后面还会介绍到）

一般来说dictionary所需存储空间较小，可存在内存中，有各种数据结构可以用来对dictionary进行高效查询（跳表、hash表、trie树等等）。postings则可能会很大，会以一种紧凑且连续的方式有序地存储在磁盘中，从而减少postings所需存储空间大小和读盘时间，而且往往都会涉及到对postings以一种压缩方式进行压缩进一步减少占用的磁盘空间


### 物理结构
lucene实际存储postings时中采用是一种分段的结构(segmented-architecture)，即将索引划分为多个segment（segement数量是动态变化的），每个segment也是一个standalone的索引，包含多个文件名为"_X.<ext>"形式的多个文件，每个文件都是postings的一部分，后缀ext标识它是倒排表的哪个部分。在检索的时候，每个segment都需要被访问一篇，最后汇总得到最终的结果。

更多关于lucene的分段结构索引，以及为什么采用这种结构的内容见[wiki](占坑待填)（Logarithmic merging）

前面也提到，es将索引划分成shards，每个shard也就是lucene里的针对部分文档的一个完整索引，所有shard构成了整个文档集的索引。每个shard可能有多个副本，副本间完全一致，shard的所有副本均匀分布在集群的各个节点上，以做到负载均衡和高可用。

# Indexing
建索引是后续检索的基础，合适的索引是后续正确、高效检索的前提。无论是lucene还是es都提供给用户关于如果建索引非常灵活的配置和选项，下面先介绍下建索引的主要步骤：

 - 获取需要被索引的数据，数据可以来自网页(web搜索引擎)、数据库、或者一些应用数据
 - 对数据内容进行tokenize，即分词，英文分词相对来说很简单，中文分词器往往需要语料库，涉及到贝叶斯、统计学等理论
 - 对分词后产生的tokens进行一些语言学上或者一些额外辅助性的处理，生成terms。比如词根化、去掉单复数、去掉stop words、生成同义词、生成token的k-grams terms（方便模糊查询）等等
 - 最后，将产生的terms和document关系映射写入索引中

其中，第一步的行为完全由使用者来控制，在我们的应用中，数据完全来源自数据库，通过订阅kafka的促销数据变动通知将活动数据从数据库中读出再写入es中。针对第二、三步，lucene和es已经它们的一些插件提供了大量的组件和配置供用户选择来满足各种各样的检索需求；第四步则对于用户来说可以视为一个黑盒子，用户可以在不了解任何索引构建算法的前提下很好的了解和使用lucene和es（对这部分感兴趣的话可以读读An Introduction to Information Retrieval的第四章index construction）

## es的分布式document存储
上面介绍了文档被索引的过程，没有涉及到任何分布式处理的内容，这节简单介绍下es中文档创建、按id获取等请求处理流程。

### shard选择
当一个文档创建请求过来时，es按照这个公式来选择这个文档存放的shard：
shard = hash(routing) % number_of_primary_shards

其中，routing是请求时传入的参数，可以是任意字符串值，默认是文档的id。number_of_primary_shards是这个es索引配置的总shards数量。也是由于这个原因，一旦索引的number_of_primary_shards配置确定后就不能再改变了，如果需要改变只能重建整个索引。

### 一个典型的文档创建请求处理过程
假设es集群有三个节点，shard数为2，replica数为2，shard和replica的分布如下图所示：
![集群索引分布](/public/fig/cluster.png)

假设node1接受到client的文档创建请求，此时，node1被称为协调节点(或者请求节点)，那么会按照如下步骤处理：

- node1根据doc id决定这个doc属于哪个shard，在本例子中是shard0，那么转发给shard0的主副本所在节点node3
- node3在shard0上执行创建文档并索引请求，成功后将请求并行转发给shard0的两个replica所在节点，分别是node1和node2，当所有副本都响应执行成功后，node3会给协调节点node1响应最终的结果
- node1收到node3的响应后，自己再返回结果给client

在这个例子中，当client收到成功响应时，文档已经所有的shard成功索引。数据已经安全可靠地存储在集群中。当然，es也提供了几个选项用于在写入性能和数据安全性之间进行tradeoff。

- replication: 默认值是sync，意为shard的主副本等待所有的副本响应成功后再返回结果，就如例子中描述的那样。当值为async时，一旦主副本完成文档写入就直接返回了，不等待其他副本的执行结果。
- consistency: 默认情况下，只有集群中可用的shard副本数量超过半数（quorum）时，es才接受写请求，为了保证副本间数据的一致性。consitency的取值有one(只有shard的主副本可用就接受写请求)、all(只有当shard所有的副本都可用时才接受写请求)、quorum（可用数量过半才接受写请求）
- timeout: 如果某个shard没有足够的副本可用怎么办，es就只能等，timeout控制这个等待时间的参数，默认值是1mintue

### 一个典型的文档按id获取请求处理过程
还是以上图的集群为例，client向node1发起了一个按id的文档获取请求，处理如下：

- node1通过id来确定文档属于的shard，比如是shard0，然后发现shard0在三个节点上都有副本，默认情况下将以round-robin方式在副本间进行负载均衡，比如转发到node2节点
- node2取出文档返回给node1，node1再返回给client

可能存在的情况是，文档刚刚被索引，还只存在于主副本，从副本都还没有，此时replica返回文档不存在，只能最终由主副本响应结果。

## 进一步了解索引
在这节中，我将尝试回答如下几个问题：为什么lucene采用这种分段结构索引、什么是近实时检索、为什么按照id进行CURD是实时的。

### 分段结构索引
如果索引内容一旦构建完成后不发生什么变动，那么世界很美好，有操作系统的filesystem cache在，大部分对索引的访问都会命中缓存，意味着这种索引是缓存友好的。此外，lucene里的filter cache也将持续有效，用到filter的检索也会很开心。然而实际应用中索引文件很少能够保持不变。lucene为了能做到动态索引的同时还尽可能获得索引不变性带来的好处，而引入分段索引。

简单地来说：lucene把整个文档集的索引分为多个段，每个段可能只是部分文档的索引倒排表。段和段之间会存在索引合并。段的组织方式类似于二项堆结构，即段大小以指数数量级递增，段合并发生在当索引中某一数量级大小的段文件存在两个时，此时这两个段文件将合并成一个更大的段。这样做带来的好处就是大段文件变动较少，还可以有效利用filesystem cache。

实际上，lucene的索引不仅包括多个segment文件，还包括一个commit point文件，commit point文件中列出所有的segments。处理读请求时会依次访问这些segments，然后汇总结果返回。任何写请求在写入segment文件前，会先写到内存中的一个index buffer中，结构如下图所示。
![index](/public/fig/index.png)

只有当doc被写入segment文件后才能生效、被检索到，而index buffer只有在满了或者使用者通过api手动刷入时才能同步到文件中，意味着刚写入索引的文档不能立即被检索到（还是在index buffer中），所幸es提供了refresh api，通过它可以把index buffer的数据写入操作系统内核的文件缓存中（write系统调用），此时这些内容就可以被检索到了。refresh是相对于提交索引(commit)而言轻量级的操作，默认情况下，es的每个shard都会每秒自动refresh一次，这也是为什么说es是近实时索引的。如果你很在意索引的实时性的话，也可以在写入文档后，立即发起一次refresh请求，但需要注意这也是有一定性能开销的。自动refresh的行为还可以调整：可以在重建索引时关闭，建好上线后再打开，以缩短重建索引的时间。

除了每秒refresh之外，为了避免节点故障（比如断电）导致的索引数据丢失，我们还需要定期提交索引(commit)，即将index buffer和filesystem cache中的数据刷到磁盘中(fsync系统调用)、同时写commit point文件，这样节点重启后也能在这个commit point处恢复过来。但是在两次commit之间如果发生故障怎么办，我们肯定也不希望数据会丢失，ES通过translog机制来保障索引数据安全。

### translog机制
ES会将每个写操作都先记入translog中，然后在写入index buffer。refresh请求会清空index buffer，写入数据到filesystem cache中，但是不会清空translog，当translog大小到达一定值时，会触发es进行一次提交索引操作，此时所有索引内容被刷入磁盘中、更新commit point文件、清空translog。translog保证了还没有从page cache sync到disk中的数据不会因为机器重启而丢失，因为当机器重启时先从最近的一个commit point恢复，然后再重放translog中所有变动就使得索引状态恢复到和重启前一致了。

translog还有一个作用，就是提供了按照id来CRUD文档时的实时性。当通过document id来取、更新、删除doc时，先check下translog，看看有没有最近的变动，然后再从对应的segment文件中取出对应的doc，也就是说，按照id来取文件的结果是实时的。

那么还有一个问题：translog本身的数据安全怎么来保证？我们知道，translog的内容如果不通过fsync持久化到磁盘中的话，在机器重启后也会丢失。默认情况下，translog的内容会在每5秒fsync一次，看上去我们依然可能丢失最多5s内的translog数据。所幸的是，translog只是一个更巨大系统的一部分。前面也提到一个典型的写请求处理过程中只有当所有的主从副本都写入成功之后这次请求算完成，意味着同一个shard也有多个translog副本，如何保证translog数据安全和一致性是分布式一致性协议(distributed consensus algorithm)所解决的问题，是个非常大话题，学界和工业界也存在很多优秀的工作用来解决这个问题(从(multi-)paxos到raft、zk的zab等等)，es貌似也是自己搞了一套。

### 简单谈谈分词(Analysis)
这里所说的分词其实包含两个步骤：

- 先对文本内按照配置的tokenizer进行tokenize，tokenizer的作用就是将原始内容切成一个个token
- 再对每个token按照配置的token filters进行处理，可能会有各种filter，每种filter的职责也不尽相同，比如有的可能是为了去掉stop words、有的是为了生成同义词、有的是为了对单词去掉单复数前后缀等等、有的是为了支持特殊的检索需求，总之非常丰富。

tokenizer结合token filters打包到一起就是所谓的analyzer，es基于lucene提供了几种默认的
analyzer供选择，如果有特殊需求的话，还可以自定义analyzer，也非常简单。

# Searching
search的内容就非常之丰富了，es基于lucene提供了各种各样的query和filter来满足各种各样的检索需求。下面以典型的full text query——match query为例介绍下query在lucene中的处理过程。
 
比如，我们现在需要在按照活动的title“商家自助促销”来检索促销活动数据，通过es的api向集群发起一个match query的请求，先忽略分布式处理的内容，每个shard的执行步骤如下：

 - 对query的内容“商家自助促销”按照title field预先配置的analyzer或者query时指定的analyzer进行分词和相关处理，得到一堆terms，比如”商家“、“自助”、”促销“等等，生成的terms由所使用的analyzer决定
 - 拿得到的这些terms去倒排表中查出所有相关的文档
 - 对取出的文档按照默认的TF-IDF打分规则进行算分，得到按照score排序的文档集返回

本节将先从lucene的排序模型开始展开，然后在介绍一些常见的query和filter，再谈谈es分布式searching。

## TF-IDF Ranking
我们知道boolean query的话只需要按terms查出满足条件的文档就行可以了，然而实际应用我们往往还需要对结果集按照查询相关性进行排序，表达相关性最常用的方法就是基于所谓的vector space model，它将query string和文档内容都视作以query中出现的所有term为基的向量，然后计算两个向量的余弦距离，距离越近表示越相关。向量权重内容就是由TF-IDF模型来确定的，它的思想很简单：TF（term frequency）表达的是term在query string或者对应的一篇文档内容中的出现频率，意味着在query string或者对应的文档内容中出现次数越多，权重越高。IDF（inverse document frequency）则是和term在整个文档空间中出现频率的倒数正相关，意味在整个文档空间出现次数越多的term，权重越低。因此，这种基于TF-IDF的向量余弦相似性就可以用下面公式来计算：

$$score(q,d) = \frac{\vec{V(q)} \cdot \vec{V(d)}}{\|\vec{V(q)}\|\|\vec{V(d)}\|}$$

其中 \\(\vec{V(q)}\\) 和 \\(\vec{V(d)}\\) 分别是query q和document d的term向量，向量的每个term的分量值为:

$$tfidf_{t,d} = tf_{t,d} * idf_t$$

即term t在document d中的tf值和t的idf值的乘积。

实际上lucene在算分时并非严格遵从纯的VSM，而是基于经验对纯VSM进行了如下几个方面的改良：


- 向量单位化项\\(\|\vec{V(d)}\|\\)是有问题的，在于它没有考虑到document本身内容长度的影响。试想某个query只有一个term，而这个term在两个doc中词频是一致的，但是其中一个doc内容非常长，除了包含这个term外还包括很多其他无关的信息，另一个doc内容很短，就只有term，那么显然我们会认为后者和这个term更相关，而在纯的VSM看来，这两个doc对这个term query来说相关性是相等的。因此，lucene引入doc_len_norm(d)，即doc长度归一化项，通过它来将\\(\vec{V(d)}\\)归一化，使得doc越长，权重越低。
- 在建索引的时候，用户有时会提升一些文档的权重，因此lucene引入了doc-boost(d)项表示文档d的权重提升值（注意这里的对文档的权重提升实际上也是针对某个field而言的）
- 在检索的时，用户有时希望能够提供正对某个term的权重，因此lucene引入了query-boost(t)
- 在检索时，如果一个query中由多个terms构成，用户总是希望匹配到越多term的文档能够越位于前面，因此lucene额外引入了一个项coord-factor(q,d)，用于提升匹配到越多term的文档。

基于这些考虑，Lucene提出所谓的Lucene's Practical Scoring Function：

$$score(q,d) = coordFactor(q,d) · queryNorm(q) · docLenNorm(d) \\
\ \ \ \ \ \ \ \ · docBoost(d) · \sum_{t\ in\ q}(\ tf(t,d) · idf_{t}^2 · queryBoost(t)\ )$$

下面分别对公式中的这些项的计算做个简单的介绍：

### coordFactor
coordFactor项计算很简单，可以由下面式子得到：

$$coordFactor(q,d) = \frac{Num_m}{Num_t}$$

其中\\(Num_t\\)是query q中的总terms数，\\(Num_m\\)是文档d中匹配的terms数。

### queryNorm
queryNorm项就是VSM模型中的\\(\frac{1}{\|\vec{V(q)}\|}\\)，计算公式如下：

$$queryNorm(q) =  \frac{1}{\sqrt[2]{sumOfSquaredWeights}}$$

注意到queryNorm项与文档无关，query确定了这个项的值就确定了，不会对结果文档集的排序有什么影响，它的作用在于使得不同的query间的结果也是可以比较的（事实上，大部分时候我们都没有这个需求，所以可以忽略它。。）

### docLenNorm & docBoost
docLenNorm和docBoost项都是文档相关的，和query无关，因此在lucene的设计中，它两在每个doc写入索引时就已经计算好乘积，并且占用1个字节存储在索引文件中。统称为doc的index-time field-level Boost（即每个doc的每个field用1个字节来存储这个值）。docLenNorm计算很简单，为\\(\frac{1}{numOfTerms}\\)，就是这个doc的这个field中的term总数的倒数。docBoost默认为1，即不对doc的权重做任何改变，另外es官方也强烈反对去设置doc level的boost，所以建议忽略。

### tf
tf(t,d)项在lucene中计算采用是frequency的平方根，frequency是就是term在doc的这个field里出现的次数。tf项也是在建索引时就计算好，存储在索引文件中的

### idf
idf项在lucene中是通过如下公式计算得到：

$$idf(t) = 1 + log (\frac{numOfDocs}{docFreq_t + 1} )$$

和tf项一样，也是在建索引的时候就计算好，存储在索引中。与tf不同的是存储idf所需的空间就少很多了，因此idf在倒排索引中每个term存一个（可以存在dictionary中），而tf则需要为每个term对应的posting list中每项（每个文档）存一个。

### queryBoost
queryBoost(t)项前面也都介绍了，在检索时可以为query中的某些term提升权重，queryBoost(t)默认值也是1，即不改变权重。

### 高效算分和排序
前面详细介绍基于VSM的tf-idf算分模型，在实现上，我们一般会给query传入limit参数，lucene的做法是在内存中先建一个小根堆（大小是limit），然后按terms从倒排表中依次取出所有的匹配的文档，放入小根堆中排序，最后小根堆中元素就是score是TOP K（K=limit）的文档。

实际上，有时一个query匹配到的结果集可能非常之大，为结果集中每个文档准确地算分再排序是一件非常耗时、其没那么重要的事（大部分时候我们只需要获取前几页的结果），因此很多搜索引擎都会应用一些启发式策略来加快这一过程，代价就是取出的top K个结果不一定是严格精确的（这也是一种典型的tradeoff，以精度换时间）。基本上所有的启发式策略都会遵从这一模式：先确定一个大小为A的候选结果集，再从这个候选结果集里找出TOP K的文档返回，其中 K < A << N，N是匹配文档的总数。更具体的内容见An Introduction to Information Retrieval的第七章中efficient scoring and ranking。


## 一些常用的query和filter

### 列举一些常用的query
简单query类型除了最基本的term query之外，lucene和es还提供很多常用query的api，主要有如下：

- MatchQuery: 前面提到过，传入一个query string，先对其进行分词，得到多个term，适用于需要全文检索的field。
- TermsQuery: termQuery的升级版，支持一次性传入多个term
- RangeQuery: term范围查询，适合数值型field
- PrefixQuery: term前缀查询
- IdsQuery: 按es文档id查询
- WildcardQuery: 支持term通配符查询
- FuzzyQuery: 支持term模糊查询（形似查询），举个例子，针对"big"的fuzzyQuery可能会查出来包含"pig"的文档，有趣的是fuzzyQuery在打分时不仅考虑TF-IDF权重，还要考虑匹配term和原来query中的term的相似度，这个相似度通过Levenshtein distance来度量（又称“编辑距离”）。
- PhraseQuery: 支持词组查询，依次传入词组中的所有terms，以及一个slop参数。对于PhraseQuery而言，结果中都必须要出现所有的terms，而且terms的位置关系和传入的词组terms位置关系差异不能超过slop值。另外，与FuzzyQuery类似的是，结果文档中terms的位置关系与原query中terms位置关系的相似程度也会影响结果排序，这个相似度也是通过编辑距离来衡量。

除了上述常见的简单类似query外，还有很多为满足各种需求提出的复合query，最常见的就是BoolQuery。BoolQuery用于对任意多个子Query进行bool组合，可以通过三种模式来组合：
 - must: 通过must组合意味着结果集中所有文档必须匹配这个子Query
 - must_not: 意味着结果集中所有文档必须不匹配这个子Query
 - should: 意味着文档最好匹配这个子Query，而且匹配了的话子Query的score就会累加文档的总score中。

es还提供了一些其他的简单query和复合query，这里不再列举了，更多内容参考es官方文档和ElasticSearch the definitive guide一书。

### filter和结构化搜索
很多时候，我们并不需要对结果计算相关性并排序，只要把满足查询条件的结果查出来，然后可能会按照某些字段的值来做排序（es也是支持的）。filter就是解决此类问题的利器。如果说query回答的是一篇文档有多匹配查询条件问题，那么filter就是回答一篇文档是否满足查询条件问题。自然地，与query类似，filter也有TermFilter、TermsFilter、RangeFilter、以及最重要的BoolFilter，用法上基本与对应的query也类似。在性能上，filter可以说是完爆query，因为：

 - filter不需要计算结果集文档的相关性
 - 最重要的是，filter引入了filter cache，它将各个leaf filter查询的结果以bitset结构缓存在内存中，下次碰到相同的filter查询时不需要再访问倒排表，直接给出结果，多个filter的组合查询也变成了bitset集合运算，也是纯内存计算，非常高效。
 - 这些缓存在内存中的bitset还能动态更新，即当有相关的新文档被索引时，cache也能跟着增量更新，依然保持有效。

所以，建议就是能用filter的地方就取用filter。

## 分布式检索
前面介绍了lucene的单机版query执行过程，以及根据文档id的索引和获取执行过程，相对来说要简单些。但是对ES这种索引分布在集群内各个节点的情况而言，一个query在执行前无法预先知道query匹配的文档分布在哪些节点上。因此ES将检索过程划分为两步，称为"query then fetch"。 

### query phase
query阶段是由协调节点发起的，它向集群中索引的所有的shard（primary或者replica副本）广播这个query，每个shard在本地按照介绍的执行过程来执行这个query，然后将结果再汇总到协调节点中，由协调节点确定一个全局有序的结果。具体如下图所示：
![query phase](/public/fig/query.png)

- client向协调节点(图中的node3)发起query请求，node3将先在本地建立一个大小为offset + limit 大小的小根堆
- node3将向集群索引的每个shard广播query请求，每个shard所在的节点都在本地build一个offset+limit大小的堆。我们知道每个shard可能会有多个replica，因此协调节点可以在这些replica见做负载均衡
- 每个shard在本地执行query，取出TOP offset+limit排好序的结果，返回给node3，由node3将这些结果merge进行自己的小根堆中，得到全局有序的TOP offset+limit结果集。至此query phase结束

### fetch phase
其实通过query phase最后一步协调节点的merge后，得到的是所有满足条件的文档集的id，至于文档内容本身我们还需要向这些文档所在shard去fetch，于是就有了这个fetch阶段（如果我们只需要文档id，不需要具体内容的话，这步也可以省略）。具体过程如下：
![fetch phase](/public/fig/fetch.png)

- 协调节点先结果集里取出从offset到offset+limit的文档id，丢掉剩下的。然后因为它知道所有文档的分布，可以直接向涉及到的shard发起都发起一个按id的Multi GET请求
- 每个shard收到请求后，按id取出文档信息后返回给协调节点
- 当协调节点收到所有文档结果后，汇总返回给client。

通过上面的介绍我们也可以看出，这种query执行对于深分页的请求而言是不友好的，举个例子，query的offset是1000，limit是10。那么每个shard都需要先查出top 1010个结果，再汇总给协调节点，假设我们有10个shard，集群一共需要先查出10100个文档，最终返回给用户的就只有10个文档。如果确实需要高效取出大量文档可以使用es提供的scan & scroll api，具体使用见es官方文档。

另外一个问题就是，整个索引分布在各个shard上，每个shard都是一个独立索引，基于tf-idf的算分过程也是每个shard各自在本地进行的，这样导致term的idf其实是每个shard的局部值，并非严格全局的idf。在实际应用中，这其实没那么重要，因为如果文档在shard上分布的均匀的话，term在每个shard上的idf值应该都是接近一致的。如果不走运，碰到了由于局部idf导致的排序不准的话，es也提供了dfs_query_then_fetch类似的query来work aroud，这种query会比query_then_fetch多一个phase来计算query中所有term的全局idf。


# 总结
本文的内容主要提炼自三本书An Introduction to Information Retrieval、Lucene in action、elasticsearch the definitive guide，以及lucene的[java doc](https://lucene.apache.org/core/4_6_0/core/org/apache/lucene/search/similarities/TFIDFSimilarity.html)。主要是介绍了索引和检索过程中涉及到的一些基础议题，事实lucene和es功能还远不止这么多，IR领域内还有很多有意思的问题，但由于篇幅关系，这些内容就不在这赘述了，有兴趣的话可以去细读下这几本书。