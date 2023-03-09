---
title: TiCDC 源码阅读（二）TiKV CDC 模块介绍
author: ['沈泰宁']
date: 2022-12-22
summary: 本文是 TiCDC 源码解读的第二篇，将介绍 TiCDC 的重要组成部分，TiKV 中的 CDC 模块。
tags: ["TiCDC"]
---

## 内容概要

TiCDC 是一款 TiDB 增量数据同步工具，通过拉取上游 TiKV 的数据变更日志，TiCDC 可以将数据解析为有序的行级变更数据输出到下游。

本文是 TiCDC 源码解读的第二篇，将于大家介绍 TiCDC 的重要组成部分，TiKV 中的 CDC 模块。我们会围绕 4 个问题和 2 个目标展开。

1. TiKV 中的 CDC 模块是什么？
2. TiKV 如何输出数据变更事件流？
3. 数据变更事件有哪些？
4. 如何确保完整地捕捉分布式事务的数据变更事件？

希望在回答完这4个问题之后，大家能：

- 🔔 了解数据从 TiDB 写入到 TiKV CDC 模块输出的流程。
- 🗝️ 了解如何完整地捕捉分布式事务的数据变更事件。

在下面的内容中，我们在和这两个目标相关的地方会标记上 🔔 和 🗝️，以便提醒读者留意自己感兴趣的地方。

## TiKV 中的 CDC 模块是什么？

### CDC 模块的形态

从代码上看，CDC 模块是 TiKV 源码的一部分，它是用 rust 写的，在 TiKV 代码库里面；从运行时上看，CDC 模块运行在 TiKV 进程中，是一个线程，专门处理 TiCDC 的请求和数据变更的捕捉。

### CDC 模块的作用

CDC 模块的作用有两个：

1. 它负责捕捉实时写入和读取历史数据变更。这里提一下历史数据变更指已经写到 rocksdb 里面的变更。
2. 它还负责计算 resolved ts。这个 resolved ts 是 CDC 模块里面特有的概念，形式上是一个 uint64 的 timestamp。它是 TiKV 事务变更流中的 perfect watermark，perfect watermark 的详细概念参考《Streaming System》的第三章，我们可以用 resolved ts 来告知下游，也就是 TiCDC，在 tikv 上所有 commit ts 小于 resolved ts 事务都已经完整发送了，下游 TiCDC 可以完整地处理这批事务了。

### CDC 模块的代码分布

CDC 模块的代码在 TiKV 代码仓库的 `compoenetns/cdc` 和 `components/resolved_ts` 模块。我们在下图中的黑框里面用红色标注了几个重点文件。

在 `delegate.rs` 文件中有个同名的 `Delegate` 结构体，它可以认为是 Region 在 CDC 模块中的“委派”，负责处理这个 region 的变更数据，包括实时的 raft 写入和历史增量数据。

在 `endpoint.rs` 文件中有个 `Endpoint` 结构体，它运行在 CDC 的主线程中，驱动整个 CDC 模块，上面的 delegate 也是运行在整个线程中的。

`initializer.rs` 文件中的 `Initializer` 结构体负责增量扫逻辑，同时也负责 delegate 的初始化，这里的增量扫就是读取保存在 rocksdb 中的历史数据变更。

`service.rs` 文件中的 `Service` 结构体，它实现了 ChagneData gRPC 服务，运行在 gRPC 的线程中，它负责 TiKV 和 TiCDC 的 RPC 交互，同时它和 `Endpoint` 中的 `Delegate` 和 `Initializer` 也会有交互，主要是接受来自它俩的数据变更事件，然后把这些事件通过 RPC 发送给 TiCDC。

最后一个重要文件是 `resolver.rs`，它与上面的文件不太一样，在 resolve_ts 这个 component 中，里面的 `Resolver` 负责计算 resolved ts。

![1.PNG](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/1_e91f78042c.PNG)

## TiKV 如何输出数据变更事件流？

我们从端到端的角度完整地走一遍数据的写入和流出。下图概括了数据的流动，我们以数据保存到磁盘为界，红色箭头代表数据从 TiDB 写入 TiKV 磁盘的方向，蓝色箭头代表数据从 TiKV 磁盘流出到 TiCDC 的方向。

![UML 图.jpg](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/UML_bb75620add.jpg)

### TiDB -> TiKV Service
 
- txn prewrite: [Tikv::kv_prewrite(PrewriteRequest)](https://github.com/tikv/tikv/blob/v6.4.0/src/server/service/kv.rs#L242)
- txn commit: [Tikv::kv_commit(CommitRequest)](https://github.com/tikv/tikv/blob/v6.4.0/src/server/service/kv.rs#L263)

我们看下从 TiDB 指向 TiKV 的红线。我们知道数据来自 TiDB 的事务写入，对于一个正常的事务来说，TiDB 需要分两次调用 TiKV 的 gRPC 接口，分别是 kv_prewrite 和 kv_commit，对应了事务中的 prewrite 和 commit，在 request 请求中包含了要写入或者删除的 key 和它的 value，以及一些事务的元数据，比如 start ts，commit ts 等。

### TiKV Service -> Txn

- txn prewrite: [Storage::sched_prewrite(PrewriteRequest)](https://github.com/tikv/tikv/blob/v6.4.0/src/server/service/kv.rs#L2189-L2241)
- txn commit: [Storage::sched_commit(CommitRequest)](https://github.com/tikv/tikv/blob/v6.4.0/src/server/service/kv.rs#L2271-L2283)

我们再看从 gRPC 指向 Txn 的红线。它代表 RPC 请求从 gRPC 模块流到事务模块的这一步。这里相应的也有两个 API 的调用，分别是 `sched_prewrite` 和 `sched_commit`，在这两个 API 中，事务模块会对 request 做一些检查，比如检查 write conflict，计算 commit ts 等（事务的细节可以参考 TiKV 的源码阅读文章，在这里就先跳过了。）

### Txn -> Raftstore

- txn prewrite: [Engine::async_write_ext(RaftCmdRequest)](https://github.com/tikv/tikv/blob/v6.4.0/src/storage/txn/scheduler.rs#L1323)
- txn commit: [Engine::async_write_ext(RaftCmdRequest)](https://github.com/tikv/tikv/blob/v6.4.0/src/storage/txn/scheduler.rs#L1323)

事务模块到 Raftstore 的红线代表：Request 通过检查后，会被事务模块序列化成对 KV 的操作，然后被组装成 `RaftCmdRequest`。`RaftCmdRequest` 再经由 `Engine::async_commit_ext API` 被发送至 Raftstore 模块。

大家可以看到 prewrite 和 commit 都是变成了 `RaftCmdRequest`，也都是通过 `Engine::async_commit_ext` 发送到 Raftstore 模块的。这说明了什么呢？它说明了到 Engine 这一层，TiDB 的请求中的事务信息已经被“抹去”了，所有的事务信息都存到了 key 和 value 里面。

Raftstore 模块会将这些 key value 提交到 Raft Log 中，如果 Raft Log Commit 成功，Apply 线程会将这些 key 和 value 写入到 Rocksdb。（这里面的细节可以参考 [TiKV 的源码阅读文章](https://cn.pingcap.com/blog/?tag=TiKV%20源码阅读)，在这里就先跳过了。）

### Rafstore -> CDC

- RaftCmd: [CoprocessorHost::on_flush_applied_cmd_batch(Vec<RaftCmdRequest>)](https://github.com/tikv/tikv/blob/v6.4.0/components/raftstore/src/store/fsm/apply.rs#L597)
- Txn Record: [Engine::async_snapshot()](https://github.com/tikv/tikv/blob/v6.4.0/src/server/raftkv.rs#L431)
  
从这里起，数据开始流出了，从 Raftstore 到 CDC 模块有两条蓝线，对应这里的两个重要的 API，分别为 `on_flush_applied_cmd_batch` 实时数据的流出，和 `async_snapshot` 历史增量数据的流出（后面会说细节）。
  
### CDC -> gRPC -> TiCDC
  
- ChangeDataEvent: [Service::event_feed() -> ChangeDataEvent](https://github.com/tikv/tikv/blob/v6.4.0/components/cdc/src/service.rs#LL201C8-L201C18)
  
最后就是从 CDC 模块到 TiCDC 这几条蓝线了。数据进入 CDC 模块后，经过一系列转换，组装成 Protobuf message，最后交给 gRPC 通过 ChangeData service 中的 `EventFeed` 这个 RPC 发送到下游的 TiCDC。
  
### CDC 模块中的数据流动

![UML 图 (1).jpg](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/UML_1_f57939f8fc.jpg)

上图示意了数据从 Raftstore 发送到 TiCDC 模块的细节。
  
数据从 Raftstore 到 CDC 模块，可以分成两个阶段，对应两条链路：
  
- **阶段 1，增量扫**，Initializer -> Delegate。
  
  Initializer 从 Raftstore 拿一个 Snapshot，然后在 Snapshot 上读一些历史数据变更，读的范围有两个维度：
  
    1. 时间维度 `(checkpoint ts, current ts]`，checkpoint ts 可以理解成 changefeed 上的 checkpoint，current ts 代表 PD leader 上的当前时间。
  
    2. key 范围 `[start key, end key)`，一般为 region 的 start key 和 end key。
  
- **阶段 2，实时写入监听**，CdcObserver -> Delegate
  
  `CdcObserver` 实现对实时写入的监听。它运行在 Raftstore 的 Apply 线程中，只有在 TiCDC 对一个 Region 发起监听后才会启动运行。我们知道所有的数据都是通过 Apply 线程写入的，所以说 `CdcObserver` 能轻松地在第一时间把数据捕捉到，然后交给 `Delegate` 。
  
我们再看一下数据从 CDC 模块到 gRPC 的流程，大体也有两部分。第一部分是汇总增量扫和实时写入；第二部分将这些数据是从 KV 数据反序列化成包含事务信息的 Protobuf message。我们再将这些事务结构体里面的信息给提取出来，填到一个 Protobuf message 里面。
  
### Raftstore 和 TiCDC 的交互

![UML 图 (2).jpg](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/UML_2_993f60ef5a.jpg)

上图是 Raftstore 和 CDC 模块的交互时序图。第一条线是 TiCDC，第二条是 CDC 线程，第三条是 Raftsotre 线程，第四条是 Apply 线程，图中每个点都是发生在线程上的一些事件，包含发消息、收消息和进程内部的处理逻辑。在这里我们重点说 Apply 线程。
  
Apply 线程在处理 Change 这个消息的时候，它会先要把缓存在内存中的 KV 的写入给刷到 RocksDB，然后获取 RocksDB 的 Snapshot，把 Snapshot 发送给 CDC 线程。这三步是串行的，保证了 Snapshot 可以看到之前所有的写入。有了这个机制保证，我们就可以确保 CDC 模块既不漏数据，也不多数据。
  
## 数据变更事件有哪些？

![image.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_8a167cddf7.png)

数据变更事件可分为两大类，第一类是 Event；第二类是 ResolvedTs。上图是 CDC Protobuf 的简化版定义，只保留了关键的 field。我们从上到下看下这个 Protobuf 定义。
  
`EventFeed` 定义了 TiCDC 和 TiKV 之间的消息交互方式，TiCDC 在一个 RPC 上可以发起对多个 Region 的监听，TiKV 以 `ChangeDataEvent` 形式将多个 Region 的数据变更事件发送给 TiCDC。
  
`Event` 代表着是 Region 级别的数据变更事件，包含了至少一个用户数据变更事件或者或者 Region 元数据的变更。它们是从单条 Raft Log 翻译得到的。我们可以注意到 `Event` 被 `repeat` 修饰了，也就是它可能包含了一个 region 多个数据变更，也可能包含多个不同 region 的数据变更。
  
`Entries` 包含了多个 `Row`。因为在 `oneof` 里面不能出现 `repeated` ，所以我们用 `Entries` 包装了下。

![image (1).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_1_f3177e681a.png)

`Row` 里面的内容非常接近 TiDB 层面的数据了，它是行级别的数据变更，包含：
1. 事务的 start ts；
2. 事务的 commit ts；
3. 事务写入的类型，Prewrite/Commit/Rollback；
4. 事务对数据的操作，`op_type` ，put 覆盖写一行和 delete 删除一行；
5. 事务写入的 key；
6. 事务写入的 value；
7. 该事务之前的 value，old value 在很多 CDC 协议上都会有体现，比如说 MySQL 的 maxwell 协议中的 “old” 字段。
  
## 如何确保完整地捕捉分布式事务的数据变更事件？
  
### 什么是“完整”？
  
我们需要定义完整是什么。在这里，“完整”的主体是 TiDB 中的事务，我们知道 TiDB 的事务会有两个写入事件，第一个是 prewrite，第二是 commit 或者 rollback。同时，TiDB 事务可能会涉及多个 key，这些有可能分布在不同的 region 上。所以，我们说“完整”地捕捉一个事务需要捕捉它涉及的**所有的 key** 和**所有的写入事件**。

![UML 图 (3).jpg](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/UML_3_42aee1488c.jpg)

上图描绘了一个涉及了三个 key 的事务，P 代表事务的 prewrite，C 代表事务的 commit，虚线代表一次捕捉。
  
前面两条虚线是不“完整”的捕捉，第一条虚线漏了所有 key 的 commit 事件，第二条虚线捕捉到了 k1 和 k2 的 prewrite 和 commit，但漏了 k3 的 commit。如果我们强行认为第二条虚线是“完整”的，则会破坏事务的原子性。
  
最后一条虚线才是“完整”的捕捉，因为它捕捉到了所有 key 的所有写入。
  
### 如何确认已经“完整”？
  
确认“完整”的方法有很多种，最简单的办法就是--等。一般来说，只要我们等的时间足够长，比如等一轮 GC lifetime，我们也能确认完整。但是这个办法会导致 TiCDC 的 RPO 不达标。
  
![UML 图 (4).jpg](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/UML_4_2ed55abe00.jpg)

上图最后两条虚线是两次“完整”的捕捉，假如第四条线十年之后才产生的，显然它对我们来说是没有意义的。第四条虽然是“完整”的，但是不是我们想要的。所以我们需要一种机制能够尽快地告知我们已经捕捉完整了，也就是图中第三条虚线，在时间上要尽可能地靠近最后一个变更的捕捉。那这个机制的话就是前面提到的 resolved ts。
  
### ResolvedTs 事件及性质

![image (2).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_2_370f9e1f45.png)

ResolvedTs 在 Protobuf 中的定义比较简单，一个 Region ID 数组和一个 resolved ts。它记录了**一批** Region 中**最小的** resolved ts，会混在数据变更事件流中发送给 TiCDC。从 resolved ts 事件生成的时候开始，TiDB 集群就不会产生 commit ts 小于 resolved ts 的事务了。从而 TiCDC 收到这个事件之后，便能确认这些 Region 上的数据变更事件的完整性了。
  
### resolved ts 的计算

![image (3).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_3_abee7d0464.png)

Resolved ts 的计算逻辑在 resolver.rs 文件中，可以用简单三行伪代码表示：
  
- 第一行，它要从 PD 那边取一个 TS，称它为 `min_ts`。
- 第二行，我们拿 `min_ts` 和 Region 中的所有 lock 的 start ts 做比较，取最小值，我们称它为 `new_resolved_ts` 。
- 我们拿 `new_resolved_ts` 和之前的 `resolved_ts` 做比较，取最大值，这就是当前时刻的 resolved ts。因为它小于所有 lock 的 start ts，所有它一定小于这些 lock 的未来的 commit ts。同时，在没有 lcok 的时候，`min_ts` 会变成 resolved ts，也是就当前时刻 PD 上最新的 ts 将会变成 resolved ts，这确保了它有足够的实时性。
  
### 数据变更事件流的例子

![UML 图 (5).jpg](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/UML_5_d5b69856c0.jpg)

上图是一个数据变更事件流的例子，也就是 gRPC EventFeed 中的 `stream ChangeDataEvent`。
  
例子中有三个事务和三个 resolved ts 事件：
  
- 第一个事务涉及了 k1 和 k2，它的 start ts 是 1， commit ts 是2。
- 第二个事务只包含了 k1 这一个 key，它的 start ts 是 3，commit ts 是 6，注意，这个事务在事件流中出现了乱序，它的 commit 先于 prewrite 出现在这条流中。
- 第三个事务包含了 k2 的一个事务，注意它只有一个 prewrite 事件，commit 事件还没发生，是一个正在进行中的一个事务。
- 第一个 resolved ts 事件中的 resolved ts 是 2，代表 commit ts 小于等于 2 的事务已经完整发送，在这个例子中可以把第一个事务安全的还原出来。
- 第二个 resolved ts 事件中的 resolved ts 是 4，这时 k1 的 commit 事件已经发送了，但是 prewrite 事件没有，4 就阻止了还原第二个事务。
- 第三个 resolved ts 事件出现后，我们就可以还原第二个事务了。
  
## 结尾
  
以上就是本文的全部内容。希望在阅读上面的内容后，读者能知道文章开头的四个问题和了解：
  
- 🔔数据从 TiDB 写入到 TiKV CDC 模块输出的流程
- 🗝️了解如何完整地捕捉分布式事务的数据变更事件
