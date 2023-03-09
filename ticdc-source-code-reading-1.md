---
title: TiCDC 源码阅读（一）TiCDC 架构概览
author: ['江宗其']
date: 2022-12-15
summary: 本篇文章是 TiCDC 源码阅读系列文章的第一期，主要叙述了 TiCDC 的目的、架构和数据同步链路，旨在让读者能够初步了解 TiCDC，为阅读其他源码文章起到一个引子的作用。
tags: ["TiCDC"]
---

这一次 TiCDC 阅读系列文章将会从源码层面来讲解 TiCDC 的基本原理，希望能够帮助读者深入地了解 TiCDC 。本篇文章是这一系列文章的第一期，主要叙述了 TiCDC 的目的、架构和数据同步链路，旨在让读者能够初步了解 TiCDC，为阅读其他源码阅读文章起到一个引子的作用。

## TiCDC 是什么？

TiCDC 是 TiDB 生态中的一个数据同步工具，它能够将上游 TiDB集群中产生的增量数据实时的同步到下游目的地。除了可以将 TiDB 的数据同步至 MySQL 兼容的数据库之外，还提供了同步至 Kafka 和 s3 的能力，支持 canal 和 avro 等多种开放消息协议供其他系统订阅数据变更。

![1.PNG](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/1_3684c70859.PNG)

上图描述了 TiCDC 在整个  TiDB 生态系统中的位置，它处于一个上游 TiDB 集群和下游其它数据系统的中间，充当了一个数据传输管道的角色。

**TiCDC 典型的应用场景为搭建多套 TiDB 集群间的主从复制，或者配合其他异构的系统搭建数据集成服务。以下将从这两方面为大家介绍：**

### 主从复制

使用 TiCDC 来搭建主从复制的 TiDB 集群时，根据从集群的使用目的，可能会对主从集群的数据一致性有不同的要求。目前 TiCDC 提供了如下两种级别的数据一致性:

![2.PNG](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/2_70eda26c9e.PNG)

- 快照一致性：通过开启 Syncpoint 功能，能够在实时的同步过程中，保证上下游集群在某个 TSO 的具备快照一致性。详细内容可以参考文档：[TiDB 主从集群的数据校验](https://docs.pingcap.com/zh/tidb/dev/upstream-downstream-diff)
- 最终一致性：通过开启 Redo Log 功能，能够在上游集群发生故障的时候，保证下游集群的数据达到最终一致的状态。详细内容可以参考文档：[使用 Redo Log 确保数据一致性](https://docs.pingcap.com/zh/tidb/dev/replicate-between-primary-and-secondary-clusters#%E7%AC%AC-5-%E6%AD%A5%E4%BD%BF%E7%94%A8-redo-log-%E7%A1%AE%E4%BF%9D%E6%95%B0%E6%8D%AE%E4%B8%80%E8%87%B4%E6%80%A7)

### 数据集成

![3.PNG](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/3_1cc9144622.PNG)

目前 TiCDC 提供将变更数据同步至 Kafka 和 S3 的能力，用户可以使用该功能将 TiDB 的数据集成进其他数据处理系统。在这种应用场景下，用户对数据采集的实时性和支持的消息格式的多样性会由较高的要求。当前我们提供了多种可供订阅的消息格式(可以参考 [配置 Kafka](https://docs.pingcap.com/zh/tidb/dev/ticdc-sink-to-kafka#sink-uri-%E9%85%8D%E7%BD%AE-kafka))，并在最近一段时间内对该场景的同步速度做了一系列优化，读者可以从之后的文章中了解相关内容。

## TiCDC 的架构

![4.PNG](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/4_67f2e4ea50.PNG)

确保数据传输的稳定性、实时性和一致性是 TiCDC 设计的核心目标。为了实现该目标，TiCDC 采用了分布式架构和无状态的服务模式，具备高可用和水平扩展的特性。想要深入了解 CDC 的架构，我们需要先认识下面这些概念：

### 系统组件

- TiKV：
  - TiKV 内部的 CDC 组件会扫描和拼装 kv change log。
  - 提供输出 kv change logs 的接口供 TiCDC 订阅。
- Capture：
  - TiCDC 运行进程，多个 capture 组成一个 TiCDC 集群。
  - 同步任务将会按照一定的调度规则被划分给一个或者多个 Capture 处理。

### 逻辑概念

- KV change log：TiKV 提供的隐藏大部分内部实现细节的的 row changed event，TiCDC 从 TiKV 拉取这些 Event。
- Owner：一种 Capture 的角色，每个 TiCDC 集群同一时刻最多只存在一个 Capture 具有 Owner 身份，它负责响应用户的请求、调度集群和同步 DDL 等任务。
- ChangeFeed：由用户启动同步任务，一个同步任务中可能包含多张表，这些表会被 Owner 划分为多个子任务分配到不同的 Capture 进行处理。
- Processor：Capture 内部的逻辑线程，一个 Capture 节点中可以运行多个 Processor。每个 Processor 负责处理 ChangeFeed 的一个子任务。
- TablePipeline：Processor 内部的数据同步管道，每个 TablePipeline 负责处理一张表，表的数据会在这个管道中处理和流转，最后被发送到下游。

### 基本特性

- 分布式：具备高可用能力，支持水平扩展。
- 实时性：常规场景下提供秒级的同步能力。
- 有序性：输出的数据行级别有序，并且提供 At least once 输出的保证。
- 原子性：提供单表事务的原子性。

## TiCDC 的生命周期

认识了以上的基本概念之后，我们可以继续了解一下 TiCDC 的生命周期。

### Owner

![5.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/5_352c474d49.png)

首先，我们需要知道，TiCDC 集群的元数据都会被存储到 PD 内置的 Etcd 中。当一个 TiCDC 集群被部署起来时，每个 Capture 都会向 Etcd 注册自己的信息，这样 Capture 就能够发现彼此的存在。接着，各个 Capture 之间会竞选出一个 Owner ，Owner 选举流程在 [cdc/capture.go](https://github.com/pingcap/tiflow/blob/master/cdc/capture/capture.go#L393) 文件的 `campaignOwner` 函数内，下面的代码删除了一些错误处理逻辑和参数设置，只保留主要的流程：

```go
for {
        // Campaign to be the owner, it blocks until it been elected.
        err := c.campaign(ctx)
        ...
        owner := c.newOwner(c.upstreamManager)
        c.setOwner(owner)
        ...
        err = c.runEtcdWorker(ownerCtx, owner,...)
        c.owner.AsyncStop()
        c.setOwner(nil)
}
```

每一个 Capture 进程都会调用该函数，进入一个竞选的 Loop 中，每个  Capture 都会持续不断地在竞选 Owner。同一时间段内只有一个 Capture 会当选，其它候选者则会阻塞在这个 Loop 中，直到上一个 Owner 退出就会有新的 Capture 当选。

最后真正的竞选是通过在 `c.campaign(ctx)` 函数内部调用 Etcd 的 `election.Campaign` 接口实现的，Etcd 保证了同一时间只有一个 Key 能够当选为 Owner。由于 Etcd 是高可用的服务，TiCDC 借助其力量实现了天然的高可用。

竞选到 Owner 角色的 Capture 会作为集群的管理者，也负责监听和响应来自用户的请求。

### ChangeFeed

TiCDC 集群启动完毕之后，用户即可使用 TiCDC 命令行工具或者 OpenAPI 创建 ChangeFeed (同步任务)。
一个 ChangeFeed 被创建之后，Owner 会负责对它进行检查和初始化，然后将以表为单位将划分为多个子任务分配给集群内的 Capture 进行同步。同步任务初始化的代码在 [cdc/owner/changefeed.go](https://github.com/pingcap/tiflow/blob/master/cdc/owner/changefeed.go#L404) 文件中。该函数的主要工作为：

1. 向上游查询该同步任务需要同步的表的 Schema 信息，为接下来调度器分配同步任务做准备。
2. 创建一个 `ddlPuller` 来拉取 DDL 。因为我们需要在同步的过程中保持多个 Capture 节点上 Schema 信息的一致，并且保证 DML 与 DDL 同步顺序。所以我们选择仅由 Owner 这个拥有 ChangeFeed 所以信息的角色同步 DDL。
3. 创建 `scheduler` ，它会负责把该同步任务拆分成多个子任务，发送给别的 Capture 进行处理。

Capture 接收到 Owner 发送过来的子任务之后，就会创建出一个 Processor 来处理它接收到的子任务，Processor 会为每张表创建出一个 TablePipeline 来同步对应的表的数据。Processor 会周期性的把每个 TablePipeline 的状态和进度信息汇报给 Owner，由 Owner 来决定是否进行调度和状态更新等操作。
总而言之，TiCDC 集群和同步任务的状态信息会在 Owner 和 Processor 之间流转，而用户需要同步的数据信息则通过 TablePipeline 这个管道传递到下游，下一个小节将会对 TablePipeline 进行讲解，理解了它，就能够理解 TiCDC 是怎么同步数据的。

### TablePipeline

![6.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/6_156ee3be14.png)

顾名思义，TablePipeline 是一个表数据流动和处理的管道。Processor 接收到一个同步子任务之后，会为每一张表创建出一个 TablePipeline，如上图所示，它主要由 Puller、Sorter、Mounter 和 Sink 构成。

- Puller： 负责拉取对应表在上游的变更数据，它隐藏了内部大量的实现细节，包括与 TiKV CDC 模块建立 gRPC 连接和反解码数据流等。
- Sorter： 负责对 Puller 输出的乱序数据进行排序，并且会把 Sink 来不及消费的数据进行落盘，起到一个蓄水池的作用。
- Mounter：根据事务提交时的表结构信息解析和填充行变更，将行变更转化为 TiCDC 能直接处理的数据结构。在这里，Mounter 需要和一个叫做 SchemaStorage 的组件进行交互，这个组件在 TiCDC 内部维护了所需表的 Schema 信息，后续会有内容对这其进行讲解。
- Sink：将 Mounter 处理过后的数据进行编解码，转化为 SQL 语句或者 Kafka 消息发送到对应下游。

这种模块化的设计方式，比较有利于代码的维护和重构。值得一提的是，如果你对 TiCDC 有兴趣，希望能够让它接入到当前 CDC 还不支持的下游系统，那么只要自己编码实现一个对应的 Sink 接口，就可以达到目的。
接下来，我们以一个具体例子的方式来讲解数据在 TiCDC 内部的流转。假设我们现在建立如下表结构：

```sql
CREATE TABLE TEST(
   NAME VARCHAR (20)     NOT NULL,
   AGE  INT              NOT NULL,
   PRIMARY KEY (NAME)
);

+-------+-------------+------+------+---------+-------+
| Field | Type        | Null | Key  | Default | Extra |
+-------+-------------+------+------+---------+-------+
| NAME  | varchar(20) | NO   | PRI  | NULL    |       |
| AGE   | int(11)     | NO   |      | NULL    |       |
+-------+-------------+------+------+---------+-------+
```

此时，在上游 TiDB 执行以下 DML：

```sql
INSERT INTO TEST (NAME,AGE)
VALUES ('Jack',20);

UPDATE TEST
SET AGE = 25
WHERE NAME = 'Jack';
```

下面我们就来看一看这两条 DML 会通过什么样的形式经过 TablePipeline ，最后写入下游。

#### Puller 拉取数据
上文中提到 Puller 负责与 TiKV CDC 组件建立 gPRC 连接然后拉取数据，这是 [/pipeline/puller.go](https://github.com/pingcap/tiflow/blob/master/cdc/processor/pipeline/puller.go#L67) 中的 Puller 大致的工作逻辑：

```go
n.plr = puller.New(... n.startTs, n.tableSpan(),n.tableID,n.tableName ...)
n.wg.Go(func() error {
   ctx.Throw(errors.Trace(n.plr.Run(ctxC)))
   ...
})
n.wg.Go(func() error {
   for {
      select {
      case <-ctxC.Done():
         return nil
      case rawKV := <-n.plr.Output():
         if rawKV == nil {
            continue
         }
         pEvent := model.NewPolymorphicEvent(rawKV)
         sorter.handleRawEvent(ctx, pEvent)
      }
   }
})
```

以上是经过简化的代码，可以看到在 `puller.New` 方法中，有两个比较重要的参数 `startTs` 和 `tableSpan()`，它们分别从时间和空间这两个维度上描述了我们想要拉取的数据范围。在 Puller 被创建出来之后，下面部分的代码分别启动了两个 goroutine，第一个负责运行 Puller 的内部逻辑，第二个则是等待 Puller 输出数据，然后把数据发给 Sorter。从 `plr.Output()` 中吐出来的数据长这个样子：

```go
// RawKVEntry notify the KV operator
type RawKVEntry struct {
   OpType OpType `msg:"op_type"`
   Key    []byte `msg:"key"`
   // nil for delete type
   Value []byte `msg:"value"`
   // nil for insert type
   OldValue []byte `msg:"old_value"`
   StartTs  uint64 `msg:"start_ts"`
   // Commit or resolved TS
   CRTs uint64 `msg:"crts"`
   ...
}
```

所以，在上游 TiDB 写入的那两条 DML 语句，在到达 Puller 的时候会是这样这样的一个数据结构

![7.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/7_39e4d6bcc1.png)

我们可以看到 Insert 语句扫描出的数据只有 value 没有 old_value，而 Update 语句则被转化为一条既有 value 又有 old_value 的行变更数据。

这样这两条数据就成功的被 Puller 拉取到了 TiCDC，但是因为 TiDB 中一张表的数据会被分散到多个 Region 上，所以 Puller 会与多个 TiKV Region Leader 节点建立连接，然后拉取数据。那实际上 TiCDC 拉取到的变更数据可能是乱序的，我们需要对拉取到的所有数据进行排序才能正确的将事务按照顺序同步到下游。

#### Sorter 排序

TablePipeline 中的 Sorter 只是一个拥有 Sorter 名字的中转站，实际上负责对数据进行排序的是它背后的 Sorter Engine，Sorter Engine 的生命周期是和 Capture 一致的，一个 Capture 节点上的所有 Processor 会共享一个 Sorter Engine。想要了解它是怎么工作的，可以阅读 [EventSorter 接口](https://github.com/pingcap/tiflow/blob/master/cdc/sorter/sorter.go#L32)和其具体实现的相关代码。

在这里，我们只需要知道数据进入 TablePipeline 中的 Sorter 后会被排序即可。假设我们现在除了上述的两条数据之外，在该表上又进行了其他的写入操作，并且该操作的数据在另外一个 Region。最终 Puller 拉到的数据如下：

![8.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/8_6fb8394f77.png)

除了数据之外，我们还可以看到 `Resolved` 的事件，这是一个在 TiCDC 系统中很重要的时间标志。当 TiCDC 收到 `Resolved` 时，**可以认为小于等于这个时间点提交的数据都已经被接收了，并且以后不会再有早于这个时间点的数据再发送下来，此时 TiCDC 可以此为界限来将收到的数据同步至下游。**

此外，我们可以看到拉取到的数据并不是按照 commit_ts 严格排序的，Sorter 会根据 commit_ts 将它们进行排序，最终得到如下的数据：

![9.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/9_10f6237293.png)

现在排好顺序的事件就可以往下游同步了，但是在这之前我们需要先对数据做一些转换，因为此时的数据是从 TiKV 中扫描出的 key-value，它们实际上只是一堆 bytes 数据，而不是下游想要消费的消息格式。

#### Mounter 解析

以上的 Event 数据从 Sorter 出来之后，Mounter 会根据其对应的表的 Schema 信息将它还原成按照表结构组织的数据。

```go
type RowChangedEvent struct {
   StartTs  uint64
   CommitTs uint64
   Table    *TableName
   ColInfos []rowcodec.ColInfo
   Columns      []*Column
   PreColumns   []*Column
   IndexColumns [][]int
   ...
}
```

可以看到，该结构体中还原出了所有的表和列信息，并且 Columns 和 PreColumns 就对应于 value 和 old_value。当 TiCDC 拿到这些信息之后我们就可以将数据继续下发至 Sink 组件，让其根据表信息和行变更数据去写下游数据库或者生产 Kafka 消息。值得注意的是，Mounter 进行的是一项 CPU 密集型工作，当一个表中所包含的字段较多时，Mounter 会消耗大量的计算资源。

#### Sink 下发数据

当 `RowChangedEvent` 被下发至 Sink 组件时，它身上已经包含了充分的信息，我们可以将其转化为 SQL 或者特定消息格式的 Kafka 消息。在上文的架构图中我们可以看到有两种 Sink，一种是接入在 Table Pipeline 中的 TableSink，另外一种是 Processor 级别共用的 ProcessorSink。它们在系统中有不同的作用：

- TableSink 作为一种 Table 级别的管理单位，缓存着要下发到 ProcessorSink 的数据，它的主要作用是方便 TiCDC 按照表为单位管理资源和进行调度
- ProcessorSink 作为真实要与数据库或者 Kafka 建立连接的 Sink 负责 SQL/Kafka 消息的转换和同步

我们再来看一看 ProcessorSink 到底如何转换这些行变更：

- 如果下游是数据库，ProcessorSink 会根据 `RowChangedEvent` 中的 Columns 和 PreColumns 来判断它到底是一个 `Insert`、`Update` 还是 `Delete` 操作，然后根据不同的操作类型，将其转化为 SQL 语句，然后再将其通过数据库连接写入下游：

```sql
/*
因为只有 Columns 所以是 Insert 语句。
*/
INSERT INTO TEST (NAME,AGE)
VALUES ('Jack',20);

/*
因为既有 Columns 且有 PreColumns 所以是 Update 语句。
*/
UPDATE TEST
SET AGE = 25
WHERE NAME = 'Jack';
```

- 如果下游是 Kafka, ProcessorSink 会作为一个 [Kafka Producer](https://docs.confluent.io/platform/current/clients/producer.html) 按照特定的消息格式将数据发送至 Kafka。 以 [Canal-JSON](https://docs.pingcap.com/tidb/v6.0/ticdc-canal-json) 为例，我们上述的 Insert 语句最终会以如下的 JSON 格式写入 Kafka：

```json
{
    "id": 0,
    "database": "test",
    "table": "TEST",
    "pkNames": [
        "NAME"
    ],
    "isDdl": false,
    "type": "INSERT",
    ...
    "ts": 2,
    "sql": "",
    ...
    "data": [
        {
            "NAME": "Jack",
            "AGE": "25"
        }
    ],
    "old": null
}
```

这样，上游 TiDB 执行的 DML 就成功的被发送到下游系统了。

## 结尾

以上就是本文的全部内容。希望在阅读完上面的内容之后，读者能够对 TiCDC 是什么？为什么？怎么实现？这几个问题有一个基本的答案。
