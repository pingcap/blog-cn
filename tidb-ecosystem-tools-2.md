---
title: TiDB Ecosystem Tools 原理解读系列（二）TiDB-Lightning Toolset 介绍
author: ['Kenny Chan']
date: 2018-12-18
summary: TiDB-Lightning Toolset 是一套快速全量导入 SQL dump 文件到 TiDB 集群的工具集，适合在上线前用作迁移现有的大型数据库到全新的 TiDB 集群。
tags: ['TiDB-Lightning','TiDB Ecosystem Tools']
---



## 简介

TiDB-Lightning Toolset 是一套快速全量导入 SQL dump 文件到 TiDB 集群的工具集，自 2.1.0 版本起随 TiDB 发布，速度可达到传统执行 SQL 导入方式的至少 3 倍、大约每小时 100 GB，适合在上线前用作迁移现有的大型数据库到全新的 TiDB 集群。

## 设计

TiDB 从 2017 年开始提供全量导入工具 [Loader](https://pingcap.com/docs-cn/tools/loader/)，它以多线程操作、错误重试、断点续传以及修改一些 TiDB 专属配置来提升数据导入速度。

![](media/tidb-ecosystem-tools-2/1.png)


然而，当我们全新初始化一个 TiDB 集群时，Loader 这种逐条 INSERT 指令在线上执行的方式从根本上是无法尽用性能的。原因在于 SQL 层的操作有太强的保证了。在整个导入过程中，TiDB 需要：

* 保证 ACID 特性，需要执行完整的事务流程。

* 保证各个 TiKV 服务器数据量平衡及有足够的副本，在数据增长的时候需要不断的分裂、调度 Regions。

这些动作确保 TiDB 整段导入的期间是稳定的，但在导入完毕前我们根本不会对外提供服务，这些保证就变成多此一举了。此外，多线程的线上导入也代表资料是乱序插入的，新的数据范围会与旧的重叠。TiKV 要求储存的数据是有序的，大量的乱序写入会令 TiKV 要不断地移动原有的数据（这称为 Compaction），这也会拖慢写入过程。

TiKV 是使用 RocksDB 以 KV 对的形式储存数据，这些数据会压缩成一个个 SST 格式文件。TiDB-Lightning Toolset使用新的思路，绕过SQL层，在线下将整个 SQL dump 转化为 KV 对、生成排好序的 SST 文件，然后直接用 [Ingestion](https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files) 推送到 RocksDB 里面。这样批量处理的方法略过 ACID 和线上排序等耗时步骤，让我们提升最终的速度。

## 架构

![](media/tidb-ecosystem-tools-2/2.png)

TiDB-Lightning Toolset 包含两个组件：tidb-lightning 和 tikv-importer。Lightning 负责解析 SQL 成为 KV 对，而 Importer 负责将 KV 对排序与调度、上传到 TiKV 服务器。

为什么要把一个流程拆分成两个程式呢？

1. Importer 与 TiKV 密不可分、Lightning 与 TiDB 密不可分，Toolset 的两者皆引用后者为库，而这样 Lightning 与 Importer 之间就出现语言冲突：TiKV 是使用 Rust 而 TiDB 是使用 Go 的。把它们拆分为独立的程式更方便开发，而双方都需要的 KV 对可以透过 gRPC 传递。

2. 分开 Importer 和 Lightning 也使横向扩展的方式更为灵活，例如可以运行多个 Lightning，传送给同一个 Importer。

以下我们会详细分析每个组件的操作原理。

### Lightning

![](media/tidb-ecosystem-tools-2/3.png)

Lightning 现时只支持经 mydumper 导出的 SQL 备份。mydumper 将每个表的内容分别储存到不同的文件，与 mysqldump 不同。这样不用解析整个数据库就能平行处理每个表。

首先，Lightning 会扫描 SQL 备份，区分出结构文件（包含 CREATE TABLE 语句）和数据文件（包含 INSERT 语句）。结构文件的内容会直接发送到 TiDB，用以建立数据库构型。

然后 Lightning 就会并发处理每一张表的数据。这里我们只集中看一张表的流程。每个数据文件的内容都是规律的 INSERT 语句，像是：

```sql
INSERT INTO `tbl` VALUES (1, 2, 3), (4, 5, 6), (7, 8, 9);  
INSERT INTO `tbl` VALUES (10, 11, 12), (13, 14, 15), (16, 17, 18);
INSERT INTO `tbl` VALUES (19, 20, 21), (22, 23, 24), (25, 26, 27);
```

Lightning 会作初步分析，找出每行在文件的位置并分配一个行号，使得没有主键的表可以唯一的区分每一行。此外亦同时将文件分割为大小差不多的区块（默认 256 MiB）。这些区块也会并发处理，让数据量大的表也能快速导入。以下的例子把文件以 20 字节为限分割成 5 块：

![](media/tidb-ecosystem-tools-2/4.png)

Lightning 会直接使用 TiDB 实例来把 SQL 转换为 KV 对，称为「KV 编码器」。与外部的 TiDB 集群不同，KV 编码器是寄存在 Lightning 进程内的，而且使用内存存储，所以每执行完一个 INSERT 之后，Lightning 可以直接读取内存获取转换后的 KV 对（这些 KV 对包含数据及索引）。

得到 KV 对之后便可以发送到 Importer。

### Importer

![](media/tidb-ecosystem-tools-2/5.png)

因异步操作的缘故，Importer 得到的原始 KV 对注定是无序的。所以，Importer 要做的第一件事就是要排序。这需要给每个表划定准备排序的储存空间，我们称之为 engine file。

对大数据排序是个解决了很多遍的问题，我们在此使用现有的答案：直接使用 RocksDB。一个 engine file 就相等于本地的 RocksDB，并设置为优化大量写入操作。而「排序」就相等于将 KV 对全写入到 engine file 里，RocksDB 就会帮我们合并、排序，并得到 SST 格式的文件。

这个 SST 文件包含整个表的数据和索引，比起 TiKV 的储存单位 Regions 实在太大了。所以接下来就是要切分成合适的大小（默认为 96 MiB）。Importer 会根据要导入的数据范围预先把 Region 分裂好，然后让 PD 把这些分裂出来的 Region 分散调度到不同的 TiKV 实例上。

最后，Importer 将 SST 上传到对应 Region 的每个副本上。然后通过 Leader 发起 Ingest 命令，把这个 SST 文件导入到 Raft group 里，完成一个 Region 的导入过程。

我们传输大量数据时，需要自动检查数据完整，避免忽略掉错误。Lightning 会在整个表的 Region 全部导入后，对比传送到 Importer 之前这个表的 Checksum，以及在 TiKV 集群里面时的 Checksum。如果两者一样，我们就有信心说这个表的数据没有问题。

一个表的 Checksum 是透过计算 KV 对的哈希值（Hash）产生的。因为 KV 对分布在不同的 TiKV 实例上，这个 Checksum 函数应该具备结合性；另外，Lightning 传送 KV 对之前它们是无序的，所以 Checksum 也不应该考虑顺序，即服从交换律。也就是说 Checksum 不是简单的把整个 SST 文件计算 SHA-256 这样就了事。

我们的解决办法是这样的：先计算每个 KV 对的 CRC64，然后用 XOR 结合在一起，得出一个 64 位元的校验数字。为减低 Checksum 值冲突的概率，我们目前会计算 KV 对的数量和大小。若速度允许，将来会加入更先进的 Checksum 方式。

## 总结和下一步计划

从这篇文章大家可以看到，Lightning 因为跳过了一些复杂、耗时的步骤使得整个导入进程更快，适合大数据量的初次导入，接下来我们还会做进一步的改进。

### 提升导入速度

现时 Lightning 会原封不动把整条 SQL 命令抛给 KV 编码器。所以即使我们省去执行分布式 SQL 的开销，但仍需要进行解析、规划及优化语句这些不必要或未被专门化的步骤。Lightning 可以调用更底层的 TiDB API，缩短 SQL 转 KV 的行程。

### 并行导入

![](media/tidb-ecosystem-tools-2/6.png)

另一方面，尽管我们可以不断的优化程序代码，单机的性能总是有限的。要突破这个界限就需要横向扩展：增加机器来同时导入。如前面所述，只要每套 TiDB-Lightning Toolset 操作不同的表，它们就能平行导进同一个集群。可是，现在的版本只支持读取本机文件系统上的 SQL dump，设置成多机版就显得比较麻烦了（要安装一个共享的网络盘，并且手动分配哪台机读取哪张表）。我们计划让 Lightning 能从网路获取 SQL dump（例如通过 S3 API），并提供一个工具自动分割数据库，降低设置成本。

### 在线导入

TiDB-Lightning 在导入时会把集群切换到一个专供 Lightning 写入的模式。目前来说 Lightning 主要用于在进入生产环境之前导入全量数据，所以在此期间暂停对外提供服务还可以接受。但我们希望支持更多的应用场景，例如恢复备份、储存 OLAP 的大规模计算结果等等，这些都需要维持集群在线上。所以接下来的一大方向是考虑怎样降低 Lightning 对集群的影响。
