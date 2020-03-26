---
title: TiKV 源码解析系列文章（一）序
author: ['唐刘']
date: 2019-01-28
summary: 在 TiDB DevCon 2019 上，我们宣布启动 TiKV 源码系列分享，帮助大家理解 TiKV 的技术细节。本文为本系列文章的第一篇。
tags: ['TiKV 源码解析','社区']
---

[TiKV](https://github.com/tikv/tikv) 是一个支持事务的分布式 Key-Value 数据库，有很多社区开发者基于 TiKV 来开发自己的应用，譬如 [titan](https://github.com/distributedio/titan)、[tidis](https://github.com/yongman/tidis)。尤其是在 TiKV 成为 [CNCF](https://www.cncf.io/) 的 [Sandbox](https://www.cncf.io/blog/2018/08/28/cncf-to-host-tikv-in-the-sandbox/) 项目之后，吸引了越来越多开发者的目光，很多同学都想参与到 TiKV 的研发中来。这时候，就会遇到两个比较大的拦路虎：

1. [Rust](https://www.rust-lang.org/) 语言：众所周知，TiKV 是使用 Rust 语言来进行开发的，而 Rust 语言的学习难度相对较高，有些人认为其学习曲线大于 C++，所以很多同学在这一步就直接放弃了。

2. 文档：最开始 TiKV 是作为 HTAP 数据库 [TiDB](https://github.com/pingcap/tidb) 的一个底层存储引擎设计并开发出来的，属于内部系统，缺乏详细的文档，以至于同学们不知道 TiKV 是怎么设计的，以及代码为什么要这么写。

对于第一个问题，我们内部正在制作一系列的 Rust 培训课程，由 Rust 作者以及 Rust 社区知名的开发者亲自操刀，预计会在今年第一季度对外发布。希望通过该课程的学习，大家能快速入门 Rust，使用 Rust 开发自己的应用。

而对于第二个问题，我们会启动 《TiKV 源码解析系列文章》以及 《Deep Dive TiKV 系列文章》计划，在《Deep Dive TiKV 系列文章》中，我们会详细介绍与解释 TiKV 所使用技术的基本原理，譬如 Raft 协议的说明，以及我们是如何对 Raft 做扩展和优化的。而 《TiKV 源码解析系列文章》则是会从源码层面给大家抽丝剥茧，让大家知道我们内部到底是如何实现的。我们希望，通过这两个系列，能让大家对 TiKV 有更深刻的理解，再加上 Rust 培训，能让大家很好的参与到 TiKV 的开发中来。

## 结构

本篇文章是《TiKV 源码解析系列文章》的序篇，会简单的给大家讲一下 TiKV 的基本模块，让大家对这个系统有一个整体的了解。

要理解 TiKV，只是了解 [https://github.com/tikv/tikv](https://github.com/tikv/tikv) 这一个项目是远远不够的，通常，我们也需要了解很多其他的项目，包括但不限于：

- [https://github.com/pingcap/raft-rs](https://github.com/pingcap/raft-rs)
- [https://github.com/pingcap/rust-prometheus](https://github.com/pingcap/rust-prometheus)
- [https://github.com/pingcap/rust-rocksdb](https://github.com/pingcap/rust-rocksdb)
- [https://github.com/pingcap/fail-rs](https://github.com/pingcap/fail-rs)
- [https://github.com/pingcap/rocksdb](https://github.com/pingcap/rocksdb)
- [https://github.com/pingcap/grpc-rs](https://github.com/pingcap/grpc-rs)
- [https://github.com/pingcap/pd](https://github.com/pingcap/pd)

在这个系列里面，我们首先会从 TiKV 使用的周边库开始介绍，然后介绍 TiKV，最后会介绍 [PD](https://github.com/pingcap/pd)。下面简单来说下我们的一些介绍计划。

### Storage Engine

TiKV 现在使用 [RocksDB](https://github.com/facebook/rocksdb) 作为底层数据存储方案。在 pingcap/rust-rocksdb 这个库里面，我们会简单说明 Rust 是如何通过 Foreign Function Interface (FFI) 来跟 C library 进行交互，以及我们是如何将 RocksDB 的 C API 封装好给 Rust 使用的。

另外，在 pingcap/rocksdb 这个库里面，我们会详细的介绍我们自己研发的 Key-Value 分离引擎 - [Titan](https://github.com/pingcap/rocksdb/tree/titan-5.15)，同时也会让大家知道如何使用 RocksDB 对外提供的接口来构建自己的 engine。

### Raft

TiKV 使用的是 [Raft](https://raft.github.io/) 一致性协议。为了保证算法的正确性，我们直接将 [etcd](https://github.com/etcd-io/etcd) 的 Go 实现 port 成了 Rust。在 pingcap/raft-rs，我们会详细介绍 Raft 的选举，Log 复制，snapshot 这些基本的功能是如何实现的。

另外，我们还会介绍对 Raft 的一些优化，譬如 pre-vote，check quorum 机制，batch 以及 pipeline。

最后，我们会说明如何去使用这个 Raft 库，这样大家就能在自己的应用里面集成 Raft 了。

### gRPC
 
TiKV 使用的是 [gRPC](https://grpc.io/) 作为通讯框架，我们直接把 Google [C gRPC](https://github.com/grpc/grpc) 库封装在 grpc-rs 这个库里面。我们会详细告诉大家如何去封装和操作 C gRPC 库，启动一个 gRPC 服务。

另外，我们还会介绍如何使用 Rust 的 [futures-rs](https://github.com/rust-lang-nursery/futures-rs) 来将异步逻辑变成类似同步的方式来处理，以及如何通过解析 protobuf 文件来生成对应的 API 代码。

最后，我们会介绍如何基于该库构建一个简单的 gRPC 服务。

### Prometheus

TiKV 使用 [Prometheus](https://prometheus.io/) 作为其监控系统， [rust-prometheus](https://github.com/pingcap/rust-prometheus) 这个库是 Prometheus 的 Rust client。在这个库里面，我们会介绍如果支持不同的 Prometheus 的数据类型（Counter，Gauge，Historgram）。

另外，我们会重点介绍我们是如何通过使用 Rust 的 Macro 来支持 Prometheus 的 Vector metrics 的。

最后，我们会介绍如何在自己的项目里面集成 Prometheus client，将自己的 metrics 存到 Prometheus 里面，方便后续分析。

### Fail

[Fail](https://github.com/pingcap/fail-rs) 是一个错误注入的库。通过这个库，我们能很方便的在代码的某些地方加上 hook，注入错误，然后在系统运行的时候触发相关的错误，看系统是否稳定。

我们会详细的介绍 Fail 是如何通过 macro 来注入错误，会告诉大家如何添加自己的 hook，以及在外面进行触发


### TiKV

TiKV 是一个非常复杂的系统，这块我们会重点介绍，主要包括：

1. Raftstore，该模块里面我们会介绍 TiKV 如何使用 Raft，如何支持 Multi-Raft。
2. Storage，该模块里面我们会介绍 Multiversion concurrency control (MVCC)，基于 [Percolator](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf) 的分布式事务的实现，数据在 engine 里面的存储方式，engine 操作相关的 API 等。
3. Server，该模块我们会介绍 TiKV 的 gRPC API，以及不同函数执行流程。
4. Coprocessor，该模块我们会详细介绍 TiKV 是如何处理 TiDB 的下推请求的，如何通过不同的表达式进行数据读取以及计算的。
5. PD，该模块我们会介绍 TiKV 是如何跟 PD 进行交互的。
6. Import，该模块我们会介绍 TiKV 如何处理大量数据的导入，以及如何跟 TiDB 数据导入工具 [TiDB Lightning](https://pingcap.com/docs-cn/stable/reference/tools/tidb-lightning/overview/) 交互的。
7. Util，该模块我们会介绍一些 TiKV 使用的基本功能库。

### PD

[PD](https://github.com/pingcap/pd) 用来负责整个 TiKV 的调度，我们会详细的介绍 PD 内部是如何使用 etcd 来进行元数据存取和高可用支持，也会介绍 PD 如何跟 TiKV 交互，如何生成全局的 ID 以及 timestamp。

最后，我们会详细的介绍 PD 提供的 scheduler，以及不同的 scheduler 所负责的事情，让大家能通过配置 scheduler 来让系统更加的稳定。

## 小结

上面简单的介绍了源码解析涉及的模块，还有一些模块譬如 [https://github.com/tikv/client-rust](https://github.com/tikv/client-rust) 仍在开发中，等完成之后我们也会进行源码解析。

我们希望通过该源码解析系列，能让大家对 TiKV 有一个更深刻的理解。当然，TiKV 的源码也是一直在不停的演化，我们也会尽量保证文档的及时更新。

最后，欢迎大家参与 TiKV 的开发。





 


