---
title: What’s New in TiDB 3.0.0 Beta.1
date: 2019-03-26
author: ['申砾']
summary: 今年 1 月份，我们发布了 TiDB 3.0.0 Beta 版本，DevCon 上也对这个版本做了介绍，经过两个月的努力，今天推出了下一个 Beta 版本 3.0.0 Beta.1。
tags: ['TiDB']
---

今年 1 月份，我们发布了 [TiDB 3.0.0 Beta 版本](https://pingcap.com/docs-cn/v3.0/releases/3.0beta/)，DevCon 上也对这个版本做了介绍，经过两个月的努力，今天推出了下一个 Beta 版本 3.0.0 Beta.1。让我们看一下这个版本相比于之前有什么改进。

## 新增特性解读

### Skyline Pruning

查询计划正确性和稳定性对于关系型数据库来说至关重要，3.0.0 Beta.1 对这部分进行了优化，引入一个叫 `Skyline Pruning` 的框架，通过一些启发式规则来更快更准确地找到最好的查询计划。详细信息可以参考 [这篇设计文档](https://github.com/pingcap/tidb/blob/master/docs/design/2019-01-25-skyline-pruning.md)。

### 日志格式统一

日志是排查程序问题的重要工具，统一且结构化的日志格式不但有利于用户理解日志内容，也有助于通过工具对日志进行定量分析。3.0.0 Beta.1 版本中对 tidb/pd/tikv 这三个组件的日志格式进行了统一，详细格式参见 [这篇文档](https://github.com/tikv/rfcs/blob/master/text/2018-12-19-unified-log-format.md)。

### 慢查询相关改进

慢查询日志是常用于排查性能问题, 在 3.0.0 Beta.1 之前慢查询日志跟其他日志混合存储在同个日志文件，并且格式为自定义的格式，不支持使用 SQL 语句或工具对其进行分析，严重影响排查问题的效率。从3.0.0 Beta.1 版本开始 TiDB 将查询日志文件输出到单独的日志文件中（默认日志文件名为 `tidb-slow.log`），用户可以系统变量或配置文件进行修改，同时兼容 MySQL 慢查询日志格式，支持使用 MySQL 生态分析工具（如 `pt-query-digest`）对慢查询日志进行分析。

除了慢查询日志之外，还增加一个虚拟表 `INFORMATION_SCHEMA.SLOW_QUERY`，可以对慢查询日志进行展示和过滤。

关于如何处理慢查询，我们后续还会专门写一篇文档进行介绍。如果你有一些好用的慢查询处理工具，也欢迎和我们进行交流。

### Window Function

MySQL 所支持的 Window Function TiDB 3.0.0 Beta.1 版本已经全都支持，这为 TiDB 向 MySQL 8 兼容迈出了一大步。想体验功能的可以下载版本尝鲜，但是不建议在生产中使用，这项功能还需要大量的测试，欢迎大家测试并反馈问题。

### 热点调度策略可配置化

热点调度是保持集群负载均衡的重要手段，但是一些场景下默认的热点调度显得不那么智能，甚至会对集群负载造成影响，所以 3.0.0 Beta.1 中增加了对负载均衡策略的人工干预方法，可以临时调整调度策略。

### 优化 Coprocessor 计算执行框架

目前已经完成 TableScan 算子，单 TableScan 即扫表性能提升 5% ~ 30%，接下来会对 IndexScan、Filter、Aggregation 等算子以及表达式计算框架进行优化。

### TiDB Lightning 性能优化

Lightning 是将大量数据导入 TiDB 的最佳方式，在特定表结构，单表数量，集群已有数量等条件下 1TB 数据导入性能提升 1 倍，时间从 6 小时降低到 3 小时以内，性能优化的脚步不会停，我们期望进一步提升性能，降低时间，期望能优化到 2 小时以内。

### 易用性相关的特性

* 使用 `/debug/zip` HTTP 接口， 可以方便地一键获取当前 TiDB 实例的信息，便于诊断问题。
* 新增通过 SQL 语句方式管理 pump/drainer 状态，简化 pump/drainer 状态管理，当前仅支持查看状态。
* 支持通过配置文件管理发送 binlog 策略, 丰富 binlog 管理方式。

更多的改进可以参见 [Release Notes](https://pingcap.com/docs-cn/v3.0/releases/3.0.0-beta.1/)，除了这些已经完成的特性之外，还有一些正在做的事情，比如 RBAC、Plan Management 都在密集开发中，希望在下一个 Beta 版本或者 RC 版本中能与大家见面。

## 开源社区

在这个版本的开发过程中，社区依然给我们很有力的支持，比如潘迪同学一直在负责 View 的完善和测试，美团的同学在推进 `Plan Management`，一些社区同学参与了 [TiDB 性能改进](https://github.com/pingcap/tidb/issues?q=is%3Aissue+is%3Aopen+label%3Atype%2Fperformance) 活动。在这里对各位贡献者表示由衷的感谢。接下来我们会开展更多的专项开发活动以及一系列面向社区的培训课程，希望能对大家了解如何做分布式数据库有帮助。

>One More Thing
>
>TiDB DevCon 2019 上对外展示的全新分析类产品 TiFlash 已经完成 Alpha 版本的开发，目前已经在进行内部测试，昨天试用了一下之后，我想说“真香”。
