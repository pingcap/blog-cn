---
title: Go Dumpling! 让导出数据更稳定
author: ['李淳竹']
date: 2021-01-04
summary: 在本文中，我们将会介绍一些 Dumpling 的进阶使用方法，帮助大家更稳定高效地导出数据。
tags: ['Dumpling']
---

>作    者：李淳竹（lichunzhu），TiDB 研发工程师。
>
>Migrate SIG Community，主要涵盖 TiDB 数据处理工具，包含 TiDB 数据备份/导入导出，TiDB 数据变更捕获，其他数据库数据迁移至 TiDB 等。

## 前言

[Dumpling](https://github.com/pingcap/dumpling) 是由 Go 语言编写的用于对数据库进行数据导出的工具。目前支持 MySQL 协议的数据库，并且针对 TiDB 的特性进行了优化。在 [Go Dumpling! 让导出数据更容易](https://mp.weixin.qq.com/s/CwhZYq2TbP72HIoorI2Isw) 中，我们简要介绍了 Dumpling 的基本功能的使用。在本文中，我们将会介绍一些 Dumpling 的进阶使用方法，帮助大家更稳定高效地导出数据。

## 导出到云盘

Dumpling 已经在 v4.0.9 支持直接导出数据到 Amazon S3 云盘，仅需将 -o 参数指定为 "s3://${Bucket}/${Folder}" 并在环境变量中配置 AWS 密钥即可。而 Dumpling 的读写并行的设计也可以减少导出到云盘时的性能损失。在测试中我们将 dumpling 运行在 AWS pod 中，开启多线程后 dumpling 导出速度可以达到 420MB/s。

更多详细使用配置参考[使用文档](https://docs.pingcap.com/zh/tidb/dev/dumpling-overview#%E5%AF%BC%E5%87%BA%E5%88%B0-amazon-s3-%E4%BA%91%E7%9B%98)。

## 导出为压缩文件

Dumpling 已经在 v4.0.9 中支持设置 `--compress gz` 将导出的 sql 文件压缩为 gz 格式，提高空间利用率，同时在上传云盘时也可以节省网络开销与存储成本。在我们的实际测试中，压缩为 gz 格式可以比直接导出 sql 格式文件节省一半以上的空间。

更多详细使用配置参考[使用文档](https://docs.pingcap.com/zh/tidb/dev/dumpling-overview#%E5%AF%BC%E5%87%BA%E5%88%B0-amazon-s3-%E4%BA%91%E7%9B%98)。

## 更完善的提示统计信息

在之前的使用过程中，有用户反馈在导出中 Dumpling 的提示信息不足，不能很好地了解 Dumpling 目前的导出状态。为此 Dumpling 在 v4.0.9 引入了更完善的信息提示机制。示例如下：

```
[2020/11/17 20:23:55.526 +08:00] [INFO] [collector.go:211] ["backup Success summary: total backup ranges: 45970, total success: 45970, total failed: 0, total take(backup time): 30m50.068578882s, total take(real time): 31m4.41078434s, total size(MB): 1797708.77, avg speed(MB/s): 971.70, total rows: 9200295024"] ["split chunk"=13.484064398s]
```

在导出任务结束后，Dumpling 会在 task summary 中显示本次导出过程的总共耗时，导出划分的 chunks，导出文件大小与 Dumpling 导出过程的平均速度等统计信息。

```
# HELP dumpling_dump_finished_rows counter for dumpling finished rows
# TYPE dumpling_dump_finished_rows counter
dumpling_dump_finished_rows 1.419506e+06
# HELP dumpling_dump_finished_size counter for dumpling finished file size
# TYPE dumpling_dump_finished_size counter
dumpling_dump_finished_size 2.92577519e+08
```

在导出过程中，Dumpling 所用端口的 /metrics 地址也可以实时查询到 dumpling 目前已导出的行数与数据大小。

Dumpling 将会始终倾听社区声音并持续改进。如果大家有更多改进意见，也欢迎大家到 [Dumpling repo](http://github.com/pingcap/dumpling) 提出 issue 与贡献 PR。

## 更完善的重试机制

在 v4.0.9 中，针对大数据导出的场景，Dumpling 加入了一系列重试机制来保证导出进程能够尽量排除网络波动的影响而顺利进行下去。主要包含了两个方面的重试机制：

1. 在上传数据到云盘时进行重试

    Dumpling 向云盘服务器发送数据很容易因为网络波动导致传输失败。Dumpling 通过为传输 client 设置合理的参数来重试传输数据的 HTTP 请求。通常情况下 Dumpling 会对可重试的错误尝试发送数据 3 次，每次重试间会有逐步增加并引入抖动的等待间隔。

2. 在数据库连接中断时进行重试

    在导出数据时数据库连接可能会不可避免地受到波动而中断。这时 Dumpling 会通过重建数据库连接的方式来尽量保证导出过程继续进行下去。

    这也会引出一个问题，Dumpling 提供了[不同的一致性选项](https://docs.pingcap.com/zh/tidb/stable/dumpling-overview#%E8%B0%83%E6%95%B4-dumpling-%E7%9A%84%E6%95%B0%E6%8D%AE%E4%B8%80%E8%87%B4%E6%80%A7%E9%80%89%E9%A1%B9)，贸然重建数据库连接将可能破坏导出快照的一致性。因此，Dumpling 针对不同的一致性配置做了不同的处理：

   - consistency 为 `snapshot` 或 `none`：

        这两种情况中，Dumpling 并没有为数据库上锁，Dumpling 会直接重建数据库连接。

   - consistency 为 `lock` 或 `flush`：

        这两种情况中，如果导出数据较大希望 Dumpling 可以重试，用户可以设置 `--transactional-consistency=false` 配置 Dumpling 在整个导出过程中持锁。这时如果发生 Dumpling 数据库连接中断的情况，Dumpling 将会首先检查锁数据库连接是否仍然工作正常，如果仍然正常 Dumpling 将会重建数据库连接使导出继续进行下去。

## 支持控制导出时的系统变量

Dumpling 支持了通过 --params 参数设置导出数据库时 session 变量，配置格式为 "character_set_client=latin1,character_set_connection=latin1"。用户可以通过一系列配置参数来实现不同的导出场景：

1. 设置导出字符集

    可以配置 --params "character_set_client=latin1,character_set_connection=latin1,character_set_results=latin1” 控制导出字符集为 latin1

2. 数据库导出内存控制

    配置 --params "tidb_distsql_scan_concurrency=5,tidb_mem_quota_query=8589934592" 减少 TiDB 导出时 scan 数据并发度与语句内存使用，从而减少 TiDB 导出时的内存使用

3. 数据库低速导出

    配置 --params "tidb-force-priority=LOW_PRIORITY,tidb_distsql_scan_concurrency=5" 可以调低导出语句执行优先级并减少 TiDB 导出时 scan 数据并发度，从而实现对数据库低影响的低速备份数据。

    上面参数列举了一些简单的 `--params` 使用场景。也欢迎大家开发出更多的使用场景，并向其他的社区小伙伴分享 Dumpling 的使用经验。

## Dumpling 后续开发计划

以下为 Dumpling 后续开发的一些计划与设想，也欢迎大家在 [Dumpling Repo](https://github.com/pingcap/dumpling) 一起交流讨论，参与开发。让我们一起让导出数据更加容易！

- [支持并行导出 cluster index 的数据](https://github.com/pingcap/dumpling/issues/75)(issue#75)

  目前 Mydumper 和 Dumpling 都可以通过指定 `rows` 参数开启表内并发，从而优化导出单个大数据表时的导出效率。它们的划分方式都是将表按照表的整数主键的从最小到最大划分为 count/rows 个区块再导出，然而这样的方式在数据的主键比较分散时导出效果会很差。尤其是 TiDB 4.0 后，开启了 auto-random 的数据表的主键将会更加离散，这势必引起数据块分布不均匀从而影响导出效率。同时 TiDB 支持了 cluster index 参数而减少回表次数，提高 TiDB 查询数据的效率，但该参数开启后 TiDB 可能会没有整数主键导致 Dumpling 无法划分并发导出的区块从而降低导出速率。

  在讨论后，相比上版本我们采用了更加精密而准确的设计，即 [TiDB 提供 TABLE SAMPLE 语句](https://github.com/pingcap/tidb/issues/20567)作为 Dumpling 划分区块的依据。该功能完成后，Dumpling 导出 TiDB 数据库的速度将会进一步提升。

- [支持 TiDB 标准错误码](https://github.com/pingcap/dumpling/issues/176)(issue#176)

  Dumpling 计划支持 [TiDB 标准错误码](https://github.com/pingcap/tidb/blob/master/docs/design/2020-05-08-standardize-error-codes-and-messages.md)，可以方便 Dumpling 进行更精细的错误处理，也有利于用户根据错误码与提示信息找到导出问题并进行修正。

- [支持导出更多种类的源数据库](https://github.com/pingcap/dumpling/issues/11)(issue#11)

  一般来说，只要需要支持的数据库有对应的 database driver 或 client，比如 Oracle 数据库的 golang driver [godror](https://github.com/godror/godror)，我们都可以轻微改造导出语句和调用的 Go 代码库后就实现该数据库的导出支持。这里也欢迎社区的小伙伴们参与，帮助 Dumpling 支持导出更多类型的数据库。


>联    系：channel #sig-migrate in the [tidbcommunity](https://join.slack.com/t/tidbcommunity/shared_invite/zt-9vpzdqh2-8LsybcK0US_nqwvfAjSU5A) slack workspace, you can join this channel through [this invitation link](https://slack.tidb.io/invite?team=tidb-community&channel=sig-migrate&ref=pingcap-community)。