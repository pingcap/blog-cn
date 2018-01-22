---
title: TiDB 1.1 Alpha Release
date: 2018-01-19
summary: 2018 年 1 月 19 日，TiDB 发布 1.1 Alpha 版。该版本对 MySQL 兼容性、SQL 优化器、系统稳定性、性能做了大量的工作。
tags: ['TiDB']
---

2018 年 1 月 19 日，TiDB 发布 1.1 Alpha 版。该版本对 MySQL 兼容性、SQL 优化器、系统稳定性、性能做了大量的工作。


## TiDB

- SQL parser
	- 兼容更多语法
- SQL 查询优化器
	- 统计信息减小内存占用
	- 优化统计信息启动时载入的时间
	- 更精确的代价估算
	- 使用 Count-Min Sketch 更精确的估算点查的代价
	- 支持更复杂的条件，更充分使用索引
- SQL 执行器
	- 使用 Chunk 结构重构所有执行器算子，提升分析型语句执行性能，减少内存占用
	- 优化 INSERT INGORE 语句性能
	- 下推更多的类型和函数
	- 支持更多的 SQL_MODE
	- 优化 `Load Data` 性能，速度提升 10 倍
	- 优化 `Use Database` 性能
	- 支持对物理算子内存使用进行统计
- Server
	- 支持 PROXY protocol

## PD

- 增加更多的 API
- 支持 TLS
- 给 Simulator 增加更多的 case
- 调度适应不同的 region size
- Fix 了一些调度的 bug

## TiKV

- 支持 Raft learner
- 优化 Raft Snapshot，减少 IO 开销
- 支持 TLS
- 优化 RocksDB 配置，提升性能
- Coprocessor 支持更多下推操作
- 增加更多的 Failpoint 以及稳定性测试 case
- 解决 PD 和 TiKV 之间重连的问题
- 增强数据恢复工具 TiKV-CTL 的功能
- region 支持按 table 进行分裂
- 支持 delete range 功能
- 支持设置 snapshot 导致的 IO 上限
- 完善流控机制

源码地址：[https://github.com/pingcap/tidb](https://github.com/pingcap/tidb)



**如今，在社区和 PingCAP 技术团队的共同努力下，TiDB 1.1 Alpha 版已发布，在此感谢社区的小伙伴们长久以来的参与和贡献。**



> 作为世界级开源的分布式关系型数据库，TiDB 灵感来自于 Google Spanner/F1，具备『分布式强一致性事务、在线弹性水平扩展、故障自恢复的高可用、跨数据中心多活』等核心特性。TiDB 于 2015 年 5 月在 GitHub 创建，同年 12 月发布 Alpha 版本，而后于 2016 年 6 月发布 Beta 版，12 月发布 RC1 版， 2017 年 3 月发布 RC2 版，6月份发布 RC3 版，8月份发布 RC4 版，并在 10 月发版 TiDB 1.0。

