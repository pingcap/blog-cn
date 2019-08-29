---
title: TiDB Best Practice
author: ['申砾']
date: 2017-07-05
summary: 本文档用于总结在使用 TiDB 时候的一些最佳实践，主要涉及 SQL 使用、OLAP/OLTP 优化技巧，特别是一些 TiDB 专有的优化开关。建议先阅读讲解 TiDB 原理的三篇文章(讲存储，说计算，谈调度)，再来看这篇文章。
tags: ['TiDB']
---

本文档用于总结在使用 TiDB 时候的一些最佳实践，主要涉及 SQL 使用、OLAP/OLTP 优化技巧，特别是一些 TiDB 专有的优化开关。
建议先阅读讲解 TiDB 原理的三篇文章([讲存储](tidb-internal-1.md)，[说计算](tidb-internal-2.md)，[谈调度](tidb-internal-3.md))，再来看这篇文章。

## 前言
数据库是一个通用的基础组件，在开发过程中会考虑到多种目标场景，在具体的业务场景中，需要根据业务的实际情况对数据的参数或者使用方式进行调整。

TiDB 是一个兼容 MySQL 协议和语法的分布式数据库，但是由于其内部实现，特别是支持分布式存储以及分布式事务，使得一些使用方法和 MySQL 有所区别。


## 基本概念
TiDB 的最佳实践与其实现原理密切相关，建议读者先了解一些基本的实现机制，包括 Raft、分布式事务、数据分片、负载均衡、SQL 到 KV 的映射方案、二级索引的实现方法、分布式执行引擎。下面会做一点简单的介绍，更详细的信息可以参考 PingCAP 公众号以及知乎专栏的一些文章。


### Raft

Raft 是一种一致性协议，能提供强一致的数据复制保证，TiDB 最底层用 Raft 来同步数据。每次写入都要写入多数副本，才能对外返回成功，这样即使丢掉少数副本，也能保证系统中还有最新的数据。比如最大 3 副本的话，每次写入 2 副本才算成功，任何时候，只丢失一个副本的情况下，存活的两个副本中至少有一个具有最新的数据。

相比 Master-Slave 方式的同步，同样是保存三副本，Raft 的方式更为高效，写入的延迟取决于最快的两个副本，而不是最慢的那个副本。所以使用 Raft 同步的情况下，异地多活成为可能。在典型的两地三中心场景下，每次写入只需要本数据中心以及离得近的一个数据中心写入成功就能保证数据的一致性，而并不需要三个数据中心都写成功。但是这并不意味着在任何场景都能构建跨机房部署的业务，当写入量比较大时候，机房之间的带宽和延迟成为关键因素，如果写入速度超过机房之间的带宽，或者是机房之间延迟过大，整个 Raft 同步机制依然无法很好的运转。


### 分布式事务
TiDB 提供完整的分布式事务，事务模型是在 [Google Percolator](https://research.google.com/pubs/pub36726.html) 的基础上做了一些优化。具体的实现大家可以参考[《Percolator 和 TiDB 事务算法》](./percolator-and-txn.md)这篇文章。这里只说两点：

+ 乐观锁

	TiDB 的事务模型采用乐观锁，只有在真正提交的时候，才会做冲突检测，如果有冲突，则需要重试。这种模型在冲突严重的场景下，会比较低效，因为重试之前的操作都是无效的，需要重复做。举一个比较极端的例子，就是把数据库当做计数器用，如果访问的并发度比较高，那么一定会有严重的冲突，导致大量的重试甚至是超时。但是如果访问冲突并不十分严重，那么乐观锁模型具备较高的效率。所以在冲突严重的场景下，推荐在系统架构层面解决问题，比如将计数器放在 Redis 中。


+ 事务大小限制

	由于分布式事务要做两阶段提交，并且底层还需要做 Raft 复制，如果一个事务非常大，会使得提交过程非常慢，并且会卡住下面的 Raft 复制流程。为了避免系统出现被卡住的情况，我们对事务的大小做了限制：

    - 单个事务包含的 SQL 语句不超过 5000 条（默认）
	- 单条 KV entry 不超过 6MB
	- KV entry 的总条数不超过 30W
	- KV entry 的总大小不超过 100MB

	在 Google 的 Cloud Spanner 上面，也有[类似的限制](https://cloud.google.com/spanner/docs/limits)。

### 数据分片
TiKV 自动将底层数据按照 Key 的 Range 进行分片。每个 Region 是一个 Key 的范围，从 StartKey 到 EndKey 的左闭右开区间。Region 中的 Key-Value 总量超过一定值，就会自动分裂。这部分用户不需要担心。


### 负载均衡
PD 会根据整个 TiKV 集群的状态，对集群的负载进行调度。调度是以 Region 为单位，以 PD 配置的策略为调度逻辑，自动完成。


### SQL on KV

TiDB 自动将 SQL 结构映射为 KV 结构。具体的可以参考[《三篇文章了解 TiDB 技术内幕 - 说计算》](./tidb-internal-2.md)这篇文档。简单来说，TiDB 做了两件事：

+ 一行数据映射为一个 KV，Key 以 `TableID` 构造前缀，以行 ID 为后缀
+ 一条索引映射为一个 KV，Key 以 `TableID+IndexID` 构造前缀，以索引值构造后缀

可以看到，对于一个表中的数据或者索引，会具有相同的前缀，这样在 TiKV 的 Key 空间内，这些 Key-Value 会在相邻的位置。那么当写入量很大，并且集中在一个表上面时，就会造成写入的热点，特别是连续写入的数据中某些索引值也是连续的(比如 update time 这种按时间递增的字段)，会在很少的几个 Region 上形成写入热点，成为整个系统的瓶颈。同样，如果所有的数据读取操作也都集中在很小的一个范围内 (比如在连续的几万或者十几万行数据上)，那么可能造成数据的访问热点。

### Secondary Index

TiDB 支持完整的二级索引，并且是全局索引，很多查询可以通过索引来优化。如果利用好二级索引，对业务非常重要，很多 MySQL 上的经验在 TiDB 这里依然适用，不过 TiDB 还有一些自己的特点，需要注意，这一节主要讨论在 TiDB 上使用二级索引的一些注意事项。

 + 二级索引是否越多越好

 二级索引能加速查询，但是要注意新增一个索引是有副作用的，在上一节中我们介绍了索引的存储模型，那么每增加一个索引，在插入一条数据的时候，就要新增一个 Key-Value，所以索引越多，写入越慢，并且空间占用越大。另外过多的索引也会影响优化器运行时间，并且不合适的索引会误导优化器。所以索引并不是越多越好。

+ 对哪些列建索引比较合适

	上面提到，索引很重要但不是越多越好，我们需要根据具体的业务特点创建合适的索引。原则上我们需要对查询中需要用到的列创建索引，目的是提高性能。下面几种情况适合创建索引：

	- 区分度比较大的列，通过索引能显著地减少过滤后的行数
	- 有多个查询条件时，可以选择组合索引，注意需要把等值条件的列放在组合索引的前面

	这里举一个例子，假设常用的查询是 `select * from t where c1 = 10 and c2 = 100 and c3 > 10`, 那么可以考虑建立组合索引 `Index cidx (c1, c2, c3)`，这样可以用查询条件构造出一个索引前缀进行 Scan。

+ 通过索引查询和直接扫描 Table 的区别

	TiDB 实现了全局索引，所以索引和 Table 中的数据并不一定在一个数据分片上，通过索引查询的时候，需要先扫描索引，得到对应的行 ID，然后通过行 ID 去取数据，所以可能会涉及到两次网络请求，会有一定的性能开销。

	如果查询涉及到大量的行，那么扫描索引是并发进行，只要第一批结果已经返回，就可以开始去取 Table 的数据，所以这里是一个并行 + Pipeline 的模式，虽然有两次访问的开销，但是延迟并不会很大。

	有两种情况不会涉及到两次访问的问题：

	- 索引中的列已经满足了查询需求。比如 Table t 上面的列 c 有索引，查询是 `select c from t where c > 10;`，这个时候，只需要访问索引，就可以拿到所需要的全部数据。这种情况我们称之为覆盖索引(Covering Index)。所以如果很关注查询性能，可以将部分不需要过滤但是需要在查询结果中返回的列放入索引中，构造成组合索引，比如这个例子： `select c1, c2 from t where c1 > 10;`，要优化这个查询可以创建组合索引 `Index c12 (c1, c2)`。
	- 表的 Primary Key 是整数类型。在这种情况下，TiDB 会将 Primary Key 的值当做行 ID，所以如果查询条件是在 PK 上面，那么可以直接构造出行 ID 的范围，直接扫描 Table 数据，获取结果。

+ 查询并发度

	数据分散在很多 Region 上，所以 TiDB 在做查询的时候会并发进行，默认的并发度比较保守，因为过高的并发度会消耗大量的系统资源，且对于 OLTP 类型的查询，往往不会涉及到大量的数据，较低的并发度已经可以满足需求。对于 OLAP 类型的 Query，往往需要较高的并发度。所以 TiDB 支持通过 System Variable 来调整查询并发度。
	- [tidb_distsql_scan_concurrency](https://pingcap.com/docs-cn/dev/reference/configuration/tidb-server/tidb-specific-variables/#tidb-distsql-scan-concurrency)

		在进行扫描数据的时候的并发度，这里包括扫描 Table 以及索引数据。

	- [tidb_index_lookup_size](https://pingcap.com/docs-cn/dev/reference/configuration/tidb-server/tidb-specific-variables/#tidb-index-lookup-size)

		如果是需要访问索引获取行 ID 之后再访问 Table 数据，那么每次会把一批行 ID 作为一次请求去访问 Table 数据，这个参数可以设置 Batch 的大小，较大的 Batch 会使得延迟增加，较小的 Batch 可能会造成更多的查询次数。这个参数的合适大小与查询涉及的数据量有关。一般不需要调整。

	- [tidb_index_lookup_concurrency](https://pingcap.com/docs-cn/dev/reference/configuration/tidb-server/tidb-specific-variables/#tidb-index-lookup-concurrency)

		如果是需要访问索引获取行 ID 之后再访问 Table 数据，每次通过行 ID 获取数据时候的并发度通过这个参数调节。

+ 通过索引保证结果顺序

	索引除了可以用来过滤数据之外，还能用来对数据排序，首先按照索引的顺序获取行 ID，然后再按照行 ID 的返回顺序返回行的内容，这样可以保证返回结果按照索引列有序。前面提到了扫索引和获取 Row 之间是并行 + Pipeline 模式，如果要求按照索引的顺序返回 Row，那么这两次查询之间的并发度设置的太高并不会降低延迟，所以默认的并发度比较保守。可以通过 [tidb_index_serial_scan_concurrency](https://pingcap.com/docs-cn/dev/reference/configuration/tidb-server/tidb-specific-variables/#tidb-index-serial-scan-concurrency) 变量进行并发度调整。

+ 逆序索引

	目前 TiDB 支持对索引进行逆序 Scan，但是速度要比顺序 Scan 慢 5 倍左右，所以尽量避免对索引的逆序 Scan。


## 场景与实践

上一节我们讨论了一些 TiDB 基本的实现机制及其对使用带来的影响，本节我们从具体的使用场景出发，谈一些更为具体的操作实践。我们以从部署到支撑业务这条链路为序，进行讨论。

### 部署

在部署之前请务必阅读 [TiDB 部署建议以及对硬件的需求](https://pingcap.com/docs-cn/dev/how-to/deploy/hardware-recommendations/)。

推荐通过 [TiDB-Ansible](https://github.com/pingcap/tidb-ansible "TiDB-Ansible")
部署 TiDB 集群，这个工具可以部署、停止、销毁、升级整个集群，非常方便易用。

具体的使用文档查看 [TiDB-Ansible 部署方案](https://pingcap.com/docs-cn/dev/how-to/deploy/orchestrated/ansible/)。非常不推荐手动部署，后期的维护和升级会很麻烦。

### 导入数据

如果有 Unique Key 并且业务端可以保证数据中没有冲突，可以在 Session 内打开这个开关： `SET @@session.tidb_skip_constraint_check=1;`

另外为了提高写入性能，可以对 TiKV 的参数进行调优，具体的文档查看 [TiKV 性能参数调优](https://pingcap.com/docs-cn/v3.0/reference/performance/tune-tikv/)。

请特别注意这个参数：

```
[raftstore]
# 默认为 true，表示强制将数据刷到磁盘上。如果是非金融安全级别的业务场景，建议设置成 false，
# 以便获得更高的性能。
sync-log = true
```

### 写入

上面提到了 TiDB 对单个事务的大小有限制，这层限制是在 KV 层面，反映在 SQL 层面的话，简单来说一行数据会映射为一个 KV entry，每多一个索引，也会增加一个 KV entry，所以这个限制反映在 SQL 层面是：

+ 单个事务包含的 SQL 语句不超过 5000 条（默认）
+ 单行数据不大于 6MB
+ 总的行数*(1 + 索引个数) < 30W
+ 一次提交的全部数据小于 100MB

> **注意**：无论是大小限制还是行数限制，还要考虑 TiDB 做编码以及事务额外 Key 的开销，在使用的时候，**建议每个事务的行数不超过 200 行，且单行数据小于 100k**，否则可能性能不佳。

建议无论是 Insert，Update 还是 Delete 语句，都通过分 Batch 或者是加 Limit 的方式限制。

在删除大量数据的时候，建议使用 `Delete * from t where xx limit 5000;` 这样的方案，通过循环来删除，用 `Affected Rows == 0` 作为循环结束条件，这样避免遇到事务大小的限制。

如果一次删除的数据量非常大，这种循环的方式会越来越慢，因为每次删除都是从前向后遍历，前面的删除之后，短时间内会残留不少删除标记(后续会被 gc 掉)，影响后面的 Delete 语句。如果有可能，建议把 Where 条件细化。举个例子，假设要删除 2017-05-26 当天的所有数据，那么可以这样做：

```
for i from 0 to 23:
    while affected_rows > 0:
        delete * from t where insert_time >= i:00:00 and insert_time < (i+1):00:00 limit 5000;
        affected_rows = select affected_rows()
```

上面是一段伪代码，意思就是要把大块的数据拆成小块删除，以避免删除过程中前面的 Delete 语句影响后面的 Delete 语句。

### 查询

看业务的查询需求以及具体的语句，可以参考 [TiDB 专用系统变量和语法](https://pingcap.com/docs-cn/dev/reference/configuration/tidb-server/tidb-specific-variables)这篇文档
可以通过 SET 语句控制 SQL 执行的并发度，另外通过 Hint 控制 Join 物理算子选择。

另外 MySQL 标准的索引选择 Hint 语法，也可以用，通过 `Use Index/Ignore Index hint` 控制优化器选择索引。

如果是个 OLTP 和 OLAP 混合类型的业务，可以把 TP 请求和 AP 请求发送到不同的 tidb-server 上，这样能够减小 AP 业务对于 TP 业务的影响。 承载 AP 业务的 tidb-server 推荐使用高配的机器，比如 CPU 核数比较多，内存比较大。

### 监控 & 日志

**Metrics 系统是了解系统状态的最佳方法，建议所有的用户都部署监控系统。**TiDB [使用 Grafana+Prometheus 监控系统状态](https://pingcap.com/docs-cn/v3.0/how-to/monitor/overview/)，如果使用 TiDB-Ansible 部署集群，那么会自动部署和配置监控系统。

监控系统中的监控项很多，大部分是给 TiDB 开发者查看的内容，如果没有对源代码比较深入的了解，并没有必要了解这些监控项。我们会精简出一些和业务相关或者是系统关键组件状态相关的监控项，放在一个独立的面板中，供用户使用。

除了监控之外，查看日志也是了解系统状态的常用方法。TiDB 的三个组件 tidb-server/tikv-server/pd-server 都有一个 `--log-file` 的参数，如果启动的时候设置了这个参数，那么日志会保存着参数所设置的文件的位置，另外会自动的按天对 Log 文件做归档。如果没有设置 `--log-file` 参数，日志会输出在 stderr 中。

### 文档

了解一个系统或者解决使用中的问题最好的方法是阅读文档，明白实现原理，TiDB 有大量的官方文档，希望大家在遇到问题的时候能先尝试通过文档或者搜索 Issue list 寻找解决方案。官方文档查看 [docs-cn](https://github.com/pingcap/docs-cn)。如果希望阅读英文文档，可以查看 [docs](https://github.com/pingcap/docs)。

其中的 [FAQ](https://pingcap.com/docs-cn/v3.0/faq/tidb/)
和[故障诊断](https://pingcap.com/docs-cn/dev/how-to/troubleshoot/cluster-setup/)章节建议大家仔细阅读。另外 TiDB 还有一些不错的工具，也有配套的文档，具体的见各项工具的 GitHub 页面。

除了文档之外，还有很多不错的文章介绍 TiDB 的各项技术细节内幕，大家可以关注下面这些文章发布渠道：

+ 公众号：微信搜索 PingCAP
+ 知乎专栏：[TiDB 的后花园](https://zhuanlan.zhihu.com/newsql)
+ [官方博客](https://pingcap.github.io/blog/)

## TiDB 的最佳适用场景

简单来说，TiDB 适合具备下面这些特点的场景：

+ 数据量大，单机保存不下
+ 不希望做 Sharding 或者懒得做 Sharding
+ 访问模式上没有明显的热点
+ 需要事务、需要强一致、需要灾备
