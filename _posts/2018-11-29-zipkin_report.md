---
layout: post
title: "微博消息业务分布式trace系统实践"
description: ""
category: 
tags: [zipkin, tracing]
---
## 前言
分布式trace系统在微服务满大街的今天被各大互联网公司广泛应用，其中zipkin作为[google dapper]()这篇论文的开源实现是小厂试水分布式trace系统的一个不错的技术选择，我们组早在16年就开始尝试将trace系统落地到微博消息系统的上下链路服务跟踪中，如今由我们组负责维护的trace系统已经在微博消息和微博视频业务中两处开花。

从业务上来说，目前微博客户端上行发消息操作整条链路已经全量接入trace系统，拉取消息链路采样接入。工具组件上，已经支持了对http接口调用、motan RPC调用 (motan是微博开源的RPC框架)、redis、mc、mysql等通用组件的trace埋点。整体trace数据的体量如下：span数：80W per second , span字节大小: 200M+ per second，在zipkin的已知用户中超过netflix，仅此于twitter。

这篇文章将主要从整体trace系统架构、存储层技术选型、埋点方案、trace数据离线分析和实时处理等几个方面介绍微博消息业务中zipkin的一些实践和思考。

<!--more-->

## dapper data model
在开始正式内容之前，先简单介绍下google dapper的data model，如果你对dapper类trace系统已经有一定了解，可以跳过下面内容。更多请见[dapper](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf)论文。

dapper模型中将整个trace链路通过一棵树来建模，树的节点是链路上经过的所有接口调用、资源访问、队列生产和消费、或者一些自定义的埋点等操作，dapper用span来表示树上的这些节点，一条链路（一颗树）的所有span都用一个traceId来标识。除此之外，span还有属性如spanId（标识一个span节点）、parentId（标识该span的父span节点）、开始时间戳、duration、span名称、类型、以及一些应用相关的annotaiton数据。

一条端到端的trace链路通常会涉及到很多个服务，产生更多的span节点，同时一个span也可能会横跨多个服务节点，最常见的比如RPC调用span，就需要横跨rpc client节点和rpc server节点。一个典型的RPC调用span如下图所示：

![rpc_span](/public/fig/rpc_span.png)

当发生跨服务节点的RPC/HTTP调用时，如何将在逻辑上分布在多个服务节点上的span关联到一起呢，这就需要服务内部和服务间传递trace span的上下文信息(trace context)，通常trace context仅少量必要的上下文传递信息，如traceId、spanId、parentId和采样标记等。对于服务内的上下文传递做法是将trace context放入Thread-Local存储中，遇到跨线程方法时需要对Thread-Local中的context信息进行拷贝。对于服务间的上下文传递比如http/rpc接口调用则需要将context信息嵌入到调用请求头中传递到下游。

zipkin的数据模型基本和dapper类似，具体见[Simplified span2 format](https://github.com/openzipkin/zipkin/issues/1499)

## 基于zipkin的trace系统整体架构
这里先给出一个zipkin官方的基于zipkin做trace完整的系统架构图

![zipkin](/public/fig/zipkin.png)

从中可以看出一个基于zipkin的trace系统主要有这几个部分：

- 客户端和服务端应用的埋点
- trace数据从应用到zipkin系统的上报和传输
- trace数据收集和存储
- trace数据查询API和UI界面

下面分别做下简单的介绍

### 客户端和服务端应用的埋点
为了方便客户端和服务端进行埋点，zipkin官方和社区提供了很多不同语言、针对常见框架的埋点工具。详细见[tracers_instrumentation](https://zipkin.io/pages/tracers_instrumentation.html)

微博平台服务主要基于java构建，我们基于zipkin的Brave API对常见的组件如servlet、motan、redis、memcache、mysql、httpClient等做了trace非侵入式逻辑织入，使用方引入这些jar包后自动获得访问这些组件时进行trace能力。对于一些如写入队列、消费队列的业务场景，由于消息的格式是业务相关的，需要埋点的话必须得侵入业务代码，同样需要侵入业务的是有些场景会希望对一些方法进行自定义的trace、或者在span中tag上一些业务相关的数据，对于这些情况需要业务直接使用Brave API进行相关埋点和注入。

### trace数据从应用到zipkin系统的上报和传输
从zipkin的架构图中可以看出，trace数据的上报是通过Reporter在应用进程内完成的，并将数据上报给Transport层，再被zipkin的Collector收集。这里的Transport组件主要有两种选择: 
- Reporter直接通过调用Zipkin的数据上报的Http接口，做到trace数据上报
- Reporter通过将trace数据写入Kafka，再由Zipkin的Kafka Collector消费

对于数据量大的应用场景建议使用kafka作为Transport，zipkin的这种in-process做法和dapper的trace数据收集上报过程不大一样，下图是dapper的做法：

![dapper](/public/fig/dapper.png)

可以看出，dapper中应用进程仅将span数据写入本地日志，然后由预装在操作系统镜像中的Dapper daemon进程负责读取日志文件交给Dapper Collectors负责trace数据的落地。

相比之外，dapper的做法对业务影响更小， zipkin的in-process上报trace数据虽然简单，但是有一定性能开销。

### trace数据收集和存储
zipkin的存储层设计上主要支持cassandra、ES作为trace数据的存储系统，两种都支持处理大规模的trace数据和按traceId、以及其他多种span属性进行trace检索。twitter在内部使用cassandra作为存储，微博在项目初期采用的ES存储方案，后来随着trace数据量的增长，ES集群写性能存在瓶颈，因此切换到cassandra存储方案，并对写cassandra做了些写优化。

这里先简单介绍下zipkin基于cassandra的存储方案，再介绍下我们对写cassandra做的一些优化，微博使用的zipkin版本是2.11.5 release版，以下的介绍也是针对这个版本。

主要有span、trace_by_service_span、span_by_service三张cassandra表，其中span表是存储所有span数据的主表，trace_by_service_span和span_by_service分别是为了能够根据service(服务名)查trace和根据服务名查span名称所建立的辅助索引表。要了解这三张表的具体设计，需要先了解cassandra的数据模型和cql相关语法（版本是3.9+），详细可参考Datastax的[cql文档](https://docs.datastax.com/en/cql/3.3/cql/ddl/ddlCQLDataModelingTOC.html)，下面我将最重要的内容简要总结下

#### CQL-based Cassandra Data modeling
- cql是cassandra类似于sql的数据库管理查询语言
- 在为数据建模时，传统关系型数据的原则是尽量normalize化，即满足关系型数据的设计范式，减少表与表之间的字段重复，为领域实体和实体间的关系建模，查询时利用join来表达实体间关系。cassandra类的nosql的原则是denormalization，即基于对数据的读写pattern来为数据建模，以数据重复带来的存储空间代价换取读写性能。
- cassandra表中每行都有一个primary key，构成如下(partition key, clustered column ...)，primary key唯一标识表中的一行数据，primary key的第一部分是cassandra用于做数据sharding/partitioning的partition key，partition key相同的row同属于一个partition，物理上也会位于同一个cassandra节点。在partition内部数据通过primary key的第二部分(clustered columns)按序聚集在一起。通过primary key/partition key去查询数据是很快的（只需访问一个partiton），默认情况下cassandra不允许使用primary key/partition key以外的column来检索整个表数据。


### zipkin的cassandra存储数据模型

```sql
CREATE TABLE IF NOT EXISTS zipkin2.span (
    trace_id            text, // when strictTraceId=false, only contains right-most 16 chars
    ts_uuid             timeuuid,
    id                  text,
    trace_id_high       text, // when strictTraceId=false, contains left-most 16 chars if present
    parent_id           text,
    kind                text,
    span                text, // span.name
    ts                  bigint,
    duration            bigint,
    l_ep                Endpoint,
    r_ep                Endpoint,
    annotations         list<frozen<annotation>>,
    tags                map<text,text>,
    shared              boolean,
    debug               boolean,
    PRIMARY KEY (trace_id, ts_uuid, id)
)
    WITH CLUSTERING ORDER BY (ts_uuid DESC)
    AND compaction = {'class': 'org.apache.cassandra.db.compaction.TimeWindowCompactionStrategy'}
    AND default_time_to_live =  604800
    AND gc_grace_seconds = 3600
    AND read_repair_chance = 0
    AND dclocal_read_repair_chance = 0.0
    AND speculative_retry = '95percentile'
    AND comment = 'Primary table for holding trace data';


ALTER TABLE zipkin2.span ADD l_service text;
ALTER TABLE zipkin2.span ADD annotation_query text; //-- can't do SASI on set<text>: ░-joined until CASSANDRA-11182

CREATE CUSTOM INDEX IF NOT EXISTS ON zipkin2.span (annotation_query) USING 'org.apache.cassandra.index.sasi.SASIIndex'
   WITH OPTIONS = {
    'mode': 'PREFIX',
    'analyzed': 'true',
    'analyzer_class':'org.apache.cassandra.index.sasi.analyzer.DelimiterAnalyzer',
    'delimiter': '░'};

CREATE CUSTOM INDEX IF NOT EXISTS ON zipkin2.span (l_service) USING 'org.apache.cassandra.index.sasi.SASIIndex'
   WITH OPTIONS = {'mode': 'PREFIX'};


```
span中主要存储zipkin的原始trace span数据，下面简单说明下几个重要的设计要点：

- primary key是(trace_id, ts_uuid, id)，其中trace_id是partition key，ts_uuid + id是clustered columns，意味着同一个traceID的span数据分布在同一个partition，位于同一个cassandra节点上，在partititon内部按照ts_uuid + id并以ts_uuid降序进行clustered。为啥这么设计呢？这里我觉得原因有如下几点：
    - 以trace_id作为partition key理解起来很简单，为了做到将同一个trace的所有span数据聚合到同一个partititon，方便以trace_id查询整个trace链路数据
    - ts_uuid是根据span的开始时间戳生成一个全局唯一的带时间戳的uuid，代码见CassandraSpanConsumer，如下：

    ``` java
     long ts_micro = s.timestampAsLong();
     if (ts_micro == 0L) ts_micro = guessTimestamp(s);
     UUID ts_uuid =
        new UUID(
          UUIDs.startOf(ts_micro != 0L ? (ts_micro / 1000L) : System.currentTimeMillis())
            .getMostSignificantBits(),
          UUIDs.random().getLeastSignificantBits());

    ```
    可以看出ts_uuid的高位是span的毫秒级开始时间戳的8字节编码，地位是随机生成的8字节，共16字节，全局唯一。高位采用span时间戳编码是为了能够根据给定的时间区间查trace，当我们利用其他secondary index查trace时通常会去指定一个时间区间，ts_uuid作为clustered column的设计允许更高效的过滤通过secondary index查出来的trace数据，这一点解释见datastax的这篇[blog](https://www.datastax.com/dev/blog/a-deep-look-to-the-cql-where-clause)的Clustering column restrictions and Secondary indices这段。另外，当使用spark去从cassandra中跑trace分析任务时，cassandra的spark connector支持[where clause push-down](https://github.com/datastax/spark-cassandra-connector/blob/master/doc/3_selection.md)特性，我们可以通过传入一个利用ts_uuid的where语句做到高效查询给定时间区间的trace数据（这一点后面还会提到)。低位采用随机数是为了使得ts_uuid能够做到唯一标识一个span，这里需要注意span的id在存储层是不能唯一标识一个span的，因为有multi-host的span存在。

    还有一个问题，有了ts_uuid为啥还要将span id作为clustered column呢，猜测可能只是想让span id作为primary key的一部分，目前作用还不是太清楚。有机会问问作者

- span中增加了两个索引字段l_service和annotation_query，其中annotation_query为了支持按照各种应用自己给span打的tag来查询trace，一个span的annotation是一个kv map结构，annotation_query字段是将这个kv map拍平后构成的一个字符串，代码如下：

```java
 static @Nullable String annotationQuery(Span span) {
    if (span.annotations().isEmpty() && span.tags().isEmpty()) return null;

    char delimiter = '░'; // as very unlikely to be in the query
    StringBuilder result = new StringBuilder().append(delimiter);
    for (Annotation a : span.annotations()) {
      if (a.value().length() > LONGEST_VALUE_TO_INDEX) continue;

      result.append(a.value()).append(delimiter);
    }

    for (Map.Entry<String, String> tag : span.tags().entrySet()) {
      if (tag.getValue().length() > LONGEST_VALUE_TO_INDEX) continue;

      result.append(tag.getKey()).append(delimiter); // search is possible by key alone
      result.append(tag.getKey() + "=" + tag.getValue()).append(delimiter);
    }
    return result.length() == 1 ? null : result.toString();
  }
```

可以看出，annotation_query是由key和key=value拼接成的字符串。l_service就是span的local_endpoint的serviceName。这两个字段都使用cassandra的SASIIndex特性来构建索引，SASIIndex索引支持对索引字段进行分词构建索引、支持一些复杂如like查询，关于cassandra索引的内容见[Cassandra Native Secondary Index Deep Dive](https://www.datastax.com/dev/blog/cassandra-native-secondary-index-deep-dive)，关于SASI索引内容可以参考[Cassandra SASI Index Technical Deep Dive](http://www.doanduyhai.com/blog/?p=2058)，这两篇blog都是出自cassandra的布道师Duy Hai Doan。有关cassandra索引的内容也十分有趣，有机会再独立开个坑写写。这里需要说明的一点是，cassandra毕竟不是一个搜索引擎，其按索引去检索功能和ES、Solr还是差一些，而且也具有很多使用限制，但在trace这个场景下是可以接受


除了span表之外，还有span_by_service和trace_by_service_span两张表，为了进行根据service查span和根据service查trace而建立的，在我们的场景了没有根据service查trace的需求，没有使用这两张表，不多做具体分析，有兴趣的可见参考这篇blog[https://kyle.ai/blog/6742.html]。


### 基于spark的cassandra中trace数据离线分析和优化

zipkin中提供了针对cassandra的离线分析服务依赖的spark job，通过cassandra-spark-connector做到让spark从canssandra中获取数据构建RDD，然后在spark中进行计算，完成分析任务。

在我们的使用场景里，trace数据规模很大，用于做离线分析的spark集群相对较小（12个24核128G内存物理机集群），直接拿zipkin的dependency spark job基本不会跑出结果，任务提交到集群几分钟后，所有spark的executor都在fullgc。原因很简单，dependency任务的逻辑是读取整个cassandra的span表到spark集群，按照traceId进行group（这里的group并非直接使用spark的groupBy，而是使用cassandra-spark-connector提供的spanBy，是一个窄依赖的spark操作），再将cassanrdarow解析得到span，构建trace树进行依赖分析。一天的trace数据就有几个T的大小，把整张span的数据都读取到spark中解析成span再计算需要非常大的集群内存。


显然没有必要将所有的trace数据都从cassandra表里读取到spark集群进行分析，因此首先一个优化就是减少从canssandra读出来的数据量，所幸cassandra-spark-connector提供了[Server-side data selection, filtering and grouping](https://github.com/datastax/spark-cassandra-connector/blob/master/doc/3_selection.md)功能，这里可以利用select api选择读取感兴趣的span表字段，利用where api做数据的按条件过滤，比如可以利用span表中的ts/ts_uuid字段查最近一个小时的trace数据（这里使用ts和使用ts_uuid有着非常大的性能区别，稍后再做解释）。

经过这个优化后，fullgc问题得以解决，一个小时数据还是我们的spark集群还是可以搞定的，但是整个spark job的执行的时间还是很长，原因是产生的spark task数非常之多，第一个stage的任务数有两万多个，意味着产生了两万多个spark partition。因此在进行进一步优化前需要搞清楚cassandra-spark-connector是怎么决定spark partition的。这里简单描述下这个过程，我们知道cassandra整张表的所有数据在逻辑上分布是一个由很多个vnode构成的环（如下图所示），cassandra通过对数据的partition key进行hash来决定将数据置于哪个vnode上。

![vnode](/public/fig/vnode.png)

cassandra-spark-connector在读取cassandra数据时会对整个环进行拆分，拆分成一段段均等hash token range，每个token range对应着一个spark partition，意味着同一个cassandra parition的数据肯定都会位于同一个spark parition（因为同一个cassandra parition的数据的parition key一样，hash token也一样）。那么cassandra-spark-connector是怎么控制这个拆分的呢？有两种方法：input.split.size_in_mb参数和coalesce方法

- input.split.size_in_mb这个参数（默认值是64）可以理解为我们希望spark的每个partition处理多大规模的cassandra row数据，整张表的总大小(MB)除以这个参数就是最终的spark partition数
- CassandraTableScanRDD重载了RDD的coalesce方法，可以通过它来直接设置最终的splitCount，也就是实际产生的spark partition数

默认情况下，会根据input.split.size_in_mb的默认值64M来划分spark partition，在我们的场景下，使用了where api做存储层过滤，这么做就不太合理了，因为大部分数据都被where过滤掉了，每个spark partition处理的数据会远小于期望的64M，因此直接设置splitCount是一个相对合理的选择。

那么splitCount应该怎么取值呢？这里需要一个tradeoff，splitCount参数设置的越小，最终的spark partition数就小，产生的task数也少，但是每个task负责读取的cassandra token range就会大，而cassandra-spark-connector是通过向cassandra发起一个类似`select columns.. from span where token >= begin and token <= end and other predicates ALLOW FILTERING` 的cql查询语句来读取数据的，具体代码见CassandraTableScanRDD的compute方法，组装cql语句的方法见tokenRangeToCqlQuery：

```scala
private def tokenRangeToCqlQuery(range: CqlTokenRange[_, _]): (String, Seq[Any]) = {
    val columns = selectedColumnRefs.map(_.cql).mkString(", ")
    val (cql, values) = if (containsPartitionKey(where)) {
      ("", Seq.empty)
    } else {
      range.cql(partitionKeyStr)
    }
    val filter = (cql +: where.predicates).filter(_.nonEmpty).mkString(" AND ")
    val limitClause = limitToClause(limit)
    val orderBy = clusteringOrder.map(_.toCql(tableDef)).getOrElse("")
    val quotedKeyspaceName = quote(keyspaceName)
    val quotedTableName = quote(tableName)
    val queryTemplate =
      s"SELECT $columns " +
        s"FROM $quotedKeyspaceName.$quotedTableName " +
        s"WHERE $filter $orderBy $limitClause ALLOW FILTERING"
    val queryParamValues = values ++ where.values
    (queryTemplate, queryParamValues)
  }
```

当cql语句的要查询的token range很大时，这个range的分布可能横跨多个cassandra的vnode，意味着一条语句的执行需要cassandra集群中很多个节点参与计算，对集群会产生很高的查询负载压力，很可能会导致执行超时，从而致使spark task重试，反而会增加了spark job的整体执行时间。同时通过了解这个cql语句的构成，也很容易知道在where语句里使用ts和ts_uuid过滤的区别，使用ts_uuid可以利用到这个clustered column，结合parition key的token range，相当于利用span表的primary key进行查询，性能要比使用ts过滤好很多。
在我们的查最近一个小时的trace数据场景里，经过尝试。将splitCount设置为1000这个量级时，job的整体耗时要相对最优，每个spark partition读取cassandra数据也基本不会超时，总体上几十分钟到一个小时左右就能够完成job。


对于这个分析服务依赖的任务而言，通过ts_uuid的存储层where过滤 + splitCount调参两个手段可以将spark job的整体执行时间控制在可接受范围内。但是对于一些不能有效应用存储层where过滤的场景比如就是需要分析一天的trace数据分析服务的成功率/各分位耗时，还是需要使用input.split.size_in_mb来限制单个spark partition所需处理的数据量，避免多大的partition，同时也意味着会产生很多partition。这时的一个优化思路是：能否对这么多个partition进行采样处理，即只读取一部分token range，避免读取整个token环，这么做的前提是trace数据在整个token环上是均匀分布的。比如默认的input.split.size_in_mb是64，在我们的数据规模下会产生2W个spark partition，通过按照1%的采样率对总partition数采样，做到让spark只保留200个partition。需要注意的是这并不是对RDD进行采样（这样会导致单条trace数据span的不完整），而是对partition维度(或者说是token range维度)的采样，通过阅读spark源码发现这个思路在实现上是可行的。


#### spark partition粒度采样
spark api中变更rdd的partition数有两个方法

- reparition: 需要对原parition分布的rdd进行进行shuffle重组，按照给定的parition数形成新的rdd
- coalesce: 变更RDD partition数的底层api，支持无shuffle的parition重组、支持定制partition重组方法（传入一个PartitionCoalescer）。repartition其实就是调用的带shuffle的coalesce

要实现spark partition粒度的采样，可以通过定制coalesce的PartitionCoalescer来实现，PartitionCoalescer要做是事就是对原来RDD的所有partition进行分组，每一组对应多个原来的partition，形成一个新的partition，从而减少RDD的partition数。默认的PartitionCoalescer有个很复杂的算法考虑到原partition数据的位置分布进行合理的分组形成新的partition。我们可以实现一个简单的带采样的PartitionCoalescer，做到对原RDD的partition粒度采样。实现如下：

```scala
class SampledPartitionCoalescer(val sampleRate: Double = 0.01) extends PartitionCoalescer with Serializable {

  val rnd = new scala.util.Random()

  override def coalesce(maxPartitions: Int, parent: RDD[_]): Array[PartitionGroup] = {
    val targetLen = math.min(parent.partitions.length, maxPartitions)
    val expectedGroupSize = math.max(1, (sampleRate * parent.partitions.length / targetLen).toInt)
    val groups = (1 to targetLen).map(_ => new PartitionGroup())
    for ((g, i) <- groups.zipWithIndex) {
      val start = ((i.toLong * parent.partitions.length) / targetLen).toInt
      val end = (((i.toLong + 1) * parent.partitions.length) / targetLen).toInt
      Random.shuffle((start until end).toList).take(expectedGroupSize).foreach(index => g.partitions += parent.partitions(index))
    }
    groups.toArray
  }
}
```

其中要实现的是PartitionCoalescer的coalesce方法，maxPartitions是调用rdd的coalesce传入的目标partition数，parent是原RDD。上面的这段scala代码做到事情就是将原RDD的partition分为maxPartitions组，每组内按照sampleRate对partition采样，保留剩下来的paritions构成新的partition。

在我们的实践中，通过控制sampleRate就完全做到控制spark job的执行时间，而且采样后数据也能在一定程度上反应服务整体的trace数据情况。



### 基于flink的trace数据实时处理








