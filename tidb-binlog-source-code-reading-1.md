---
title: TiDB Binlog 源码阅读系列文章（一）序
author: ['黄佳豪']
date: 2019-06-17
summary: TiDB Binlog 组件用于收集 TiDB 的 binlog，并准实时同步给下游，如 TiDB、MySQL 等。该组件在功能上类似于 MySQL 的主从复制，会收集各个 TiDB 实例产生的 binlog，并按事务提交的时间排序，全局有序的将数据同步至下游。
tags: ['TiDB Binlog 源码阅读','社区']
---

[TiDB Binlog](https://github.com/pingcap/tidb-binlog) 组件用于收集 TiDB 的 binlog，并准实时同步给下游，如 TiDB、MySQL 等。该组件在功能上类似于 MySQL 的主从复制，会收集各个 TiDB 实例产生的 binlog，并按事务提交的时间排序，全局有序的将数据同步至下游。利用 TiDB Binlog 可以实现数据准实时同步到其他数据库，以及 TiDB 数据准实时的备份与恢复。随着大家使用的广泛和深入，我们遇到了不少由于对 TiDB Binlog 原理不理解而错误使用的情况，也发现了一些 TiDB Binlog 支持并不完善的场景和可以改进的设计。

在这样的背景下，我们开展 TiDB Binlog 源码阅读分享活动，通过对 TiDB Binlog 代码的分析和设计原理的解读，帮助大家理解 TiDB Binlog 的实现原理，和大家进行更深入的交流，同时也有助于社区参与 TiDB Binlog 的设计、开发和测试。

## 背景知识

本系列文章会聚焦 TiDB Binlog 本身，读者需要有一些基本的知识，包括但不限于：

* Go 语言，TiDB Binlog 由 Go 语言实现，有一定的 Go 语言基础有助于快速理解代码。
* 数据库基础知识，包括 MySQL、TiDB 的功能、配置和使用等；了解基本的 DDL、DML 语句和事务的基本常识。
* 了解 Kafka 的基本原理。
* 基本的后端服务知识，比如后台服务进程管理、RPC 工作原理等。

总体而言，读者需要有一定 MySQL/TiDB/Kafka 的使用经验，以及可以读懂 Go 语言程序。在阅读 TiDB Binlog 源码之前，可以先从阅读 [《TiDB Binlog 架构演进与实现原理》](https://pingcap.com/blog-cn/tidb-ecosystem-tools-1/) 入手。

## 内容概要

本篇作为《TiDB Binlog 源码阅读系列文章》的序篇，会简单的给大家讲一下后续会讲哪些部分以及逻辑顺序，方便大家对本系列文章有整体的了解。

1.  初识 TiDB Binlog 源码：整体介绍一下 TiDB Binlog 以及源码，包括 TiDB Binlog 主要有哪些组件与模块，以及如何在本地利用集成测试框架快速启动一个集群，方便大家体验 Binlog 同步功能与后续可能修改代码的测试。

2.  pump client 介绍：介绍 pump client 同时让大家了解 TiDB 是如何生成 binlog 的。

3.  pump server 介绍：介绍 pump 启动的主要流程，包括状态维护，定时触发 gc 与生成 fake binlog 驱动下游。

4.  pump storage 模块：storage 是 pump 的主要模块，主要负载 binlog 的存储，读取与排序, 可能分多篇讲解。

5.  drainer server 介绍：drainer 启动的主要流程，包括状态维护，如何获取全局 binlog 数据以及 Schema 信息。

6.  drainer loader package 介绍：loader packge 是负责实时同步数据到 mysql 的模块，在 TiDB Binlog 里多处用到。

7.  drainer sync 模块介绍：以同步 mysql 为例介绍 drainer 是如何同步到不同下游系统。

8.  slave binlog 介绍：介绍 drainer 如何转换与输出 binlog 数据到 Kafka。

9.  arbiter 介绍：同步 Kafka 中的数据到下游，通过了解 arbiter，大家可以了解如何同步数据到其他下游系统，比如更新 Cache，全文索引系统等。

10.  reparo 介绍：通过了解 reparo，大家可以将 drainer 的增量备份文件恢复到 TiDB 中。

## 小结

本篇文章主要介绍了 TiDB Binlog 源码阅读系列文章的目的和规划。下一篇文章我们会从 TiDB Binlog 的整体架构切入，然后分别讲解各个组件和关键设计点。更多的源码内容会在后续文章中逐步展开，敬请期待。

最后欢迎大家参与 [TiDB Binlog](https://github.com/pingcap/tidb-binlog) 的开发。
