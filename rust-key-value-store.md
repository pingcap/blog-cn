---
title: 使用 Rust 构建分布式 Key-Value Store
author: ['唐刘']
date: 2017-11-15
summary: 构建一个分布式 Key-Value Store 并不是一件容易的事情，我们需要考虑很多的问题，首先就是我们的系统到底需要提供什么样的功能。本文将以我们开发的分布式 Key-Value TiKV 作为实际例子，来说明下我们是如何取舍并实现的。
tags: ['Rust', 'TiKV']
---

## 引子

构建一个分布式 Key-Value Store 并不是一件容易的事情，我们需要考虑很多的问题，首先就是我们的系统到底需要提供什么样的功能，譬如：

+ 一致性：我们是否需要保证整个系统的线性一致性，还是能容忍短时间的数据不一致，只支持最终一致性。

+ 稳定性：我们能否保证系统 7 x 24 小时稳定运行。系统的可用性是 4 个 9，还有 5 个 9？如果出现了机器损坏等灾难情况，系统能否做的自动恢复。

+ 扩展性：当数据持续增多，能否通过添加机器就自动做到数据再次平衡，并且不影响外部服务。

+ 分布式事务：是否需要提供分布式事务支持，事务隔离等级需要支持到什么程度。

上面的问题在系统设计之初，就需要考虑好，作为整个系统的设计目标。为了实现这些特性，我们就需要考虑到底采用哪一种实现方案，取舍各个方面的利弊等。

后面，我将以我们开发的分布式 Key-Value [TiKV](https://github.com/pingcap/tikv) 作为实际例子，来说明下我们是如何取舍并实现的。

## TiKV

TiKV 是一个分布式 Key-Value store，它使用 Rust 开发，采用 Raft 一致性协议保证数据的强一致性，以及稳定性，同时通过 Raft 的 Configuration Change 机制实现了系统的可扩展性。

TiKV 提供了基本的  KV API 支持，也就是通常的 Get，Set，Delete，Scan 这样的 API。TiKV 也提供了支持 ACID 事务的 Transaction API，我们可以使用 Begin 开启一个事务，在事务里面对 Key 进行操作，最后再用 Commit 提交一个事务，TiKV 支持 SI 以及 SSI 事务隔离级别，用来满足用户的不同业务场景。

## Rust

在规划好 TiKV 的特性之后，我们就要开始进行 TiKV 的开发。这时候，我们面临的第一个问题就是采用什么样的语言进行开发。当时，摆在我们眼前的有几个选择：

+ Go，Go 是我们团队最擅长的一门语言，而且 Go 提供的 goroutine，channel 这些机制，天生的适合大规模分布式系统的开发，但灵活方便的同时也有一些甜蜜的负担，首先就是 GC，虽然现在 Go 的 GC 越来越完善，但总归会有短暂的卡顿，另外 goroutine 的调度也会有切换开销，这些都可能会造成请求的延迟增高。

+ Java，现在世面上面有太多基于 Java 做的分布式系统了，但 Java 一样有 GC 等开销问题，同时我们团队在 Java 上面没有任何开发经验，所以没有采用。

+ C++，C++ 可以认为是开发高性能系统的代名词，但我们团队没有特别多的同学能熟练掌握 C++，所以开发大型 C++ 项目并不是一件非常容易的事情。虽然使用现代 C++ 的编程方式能大量减少 data race，dangling pointer 等风险，我们仍然可能犯错。

当我们排除了上面几种主流语言之后，我们发现，为了开发 TiKV，我们需要这门语言具有如下特性：

+ 静态语言，这样才能最大限度的保证运行性能。

+ 无 GC，完全手动控制内存。

+ Memory safe，尽量避免 dangling pointer，memory leak 等问题。

+ Thread safe，不会遇到 data race 等问题。

+ 包管理，我们可以非常方便的使用第三方库。

+ 高效的 C 绑定，因为我们还可能使用一些 C library，所以跟 C 交互不能有开销。

综上，我们决定使用 [Rust](https://www.rust-lang.org/)，Rust 是一门系统编程语言，它提供了我们上面想要的语言特性，但选择 Rust 对我们来说也是很有风险的，主要有两点：

1. 我们团队没有任何 Rust 开发经验，全部都需要花时间学习 Rust，而偏偏 Rust 有一个非常陡峭的学习曲线。

2. 基础网络库的缺失，虽然那个时候 Rust 已经出了 1.0，但我们发现很多基础库都没有，譬如在网络库上面只有 mio，没有好用的 RPC 框架，HTTP 也不成熟。

但我们还是决定使用 Rust，对于第一点，我们团队花了将近一个月的时间来学习 Rust，跟 Rust 编译器作斗争，而对于第二点，我们就完全开始自己写。

幸运的，当我们越过 Rust 那段阵痛期之后，发现用 Rust 开发 TiKV 异常的高效，这也就是为啥我们能在短时间开发出 TiKV 并在生产环境中上线的原因。

## 一致性协议

对于分布式系统来说，CAP 是一个不得不考虑的问题，因为 P 也就是 Partition Tolerance 是一定存在的，所以我们就要考虑到底是选择 C - Consistency 还是 A - Availability。

我们在设计 TiKV 的时候就决定 - 完全保证数据安全性，所以自然就会选择 C，但其实我们并没有完全放弃 A，因为多数时候，毕竟断网，机器停电不会特别频繁，我们只需要保证 HA - High Availability，也就是 4 个 9 或者 5 个 9 的可用性就可以了。

既然选择了 C，我们下一个就考虑的是选用哪一种分布式一致性算法，现在流行的无非就是 Paxos 或者 Raft，而 Raft 因为简单，容易理解，以及有很多现成的开源库可以参考，自然就成了我们的首要选择。

在 Raft 的实现上，我们直接参考的 [etcd](https://github.com/coreos/etcd) 的 Raft。etcd 已经被大量的公司在生产环境中使用，所以它的 Raft 库质量是很有保障的。虽然 etcd 是用 Go 实现的，但它的 Raft library 是类似 C 的实现，所以非常便于我们用 Rust 直接翻译。在翻译的过程中，我们也给 etcd 的 Raft fix 了一些 bug，添加了一些功能，让其变得更加健壮和易用。

现在 Raft 的代码仍然在 TiKV 工程里面，但我们很快会将独立出去，变成独立的 library，这样大家就能在自己的 Rust 项目中使用 Raft 了。

使用 Raft 不光能保证数据的一致性，也可以借助 Raft 的 Configuration Change 机制实现系统的水平扩展，这个我们会在后面的文章中详细的说明。

## 存储引擎

选择了分布式一致性协议，下一个就要考虑数据存储的问题了。在  TiKV 里面，我们会存储 Raft log，然后也会将 Raft log 里面实际的客户请求应用到状态机里面。

首先来看状态机，因为它会存放用户的实际数据，而这些数据完全可能是随机的 key - value，为了高效的处理随机的数据插入，自然我们就考虑使用现在通用的 LSM Tree 模型。而在这种模型下，RocksDB 可以认为是现阶段最优的一个选择。

RocksDB 是 Facebook 团队在 LevelDB 的基础上面做的高性能 Key-Value Storage，它提供了很多配置选项，能让大家根据不同的硬件环境去调优。这里有一个梗，说的是因为 RocksDB 配置太多，以至于连 RocksDB team 的同学都不清楚所有配置的意义。

关于我们在 TiKV 中如何使用，优化 RocksDB，以及给 RocksDB 添加功能，fix bug 这些，我们会在后面文章中详细说明。

而对于 Raft Log，因为任意 Log 的 index 是完全单调递增的，譬如 Log 1，那么下一个 Log 一定是 Log 2，所以 Log 的插入可以认为是顺序插入。这种的，最通常的做法就是自己写一个 Segment File，但现在我们仍然使用的是 RocksDB，因为 RocksDB 对于顺序写入也有非常高的性能，也能满足我们的需求。但我们不排除后面使用自己的引擎。

因为 RocksDB 提供了 C API，所以可以直接在 Rust 里面使用，大家也可以在自己的 Rust 项目里面通过 [rust-rocksdb](https://github.com/pingcap/rust-rocksdb) 这个库来使用 RocksDB。

## 分布式事务

要支持分布式事务，首先要解决的就是分布式系统时间的问题，也就是我们用什么来标识不同事务的顺序。通常有几种做法：

+ TrueTime，TrueTime 是 Google Spanner 使用的方式，不过它需要硬件 GPS + 原子钟支持，而且 Spanner 并没有在论文里面详细说明硬件环境是如何搭建的，外面要自己实现难度比较大。

+ HLC，HLC 是一种混合逻辑时钟，它使用 Physical Time 和 Logical Clock 来确定事件的先后顺序，HLC 已经在一些应用中使用，但 HLC 依赖 NTP，如果 NTP 精度误差比较大，很可能会影响 commit wait time。

+ TSO，TSO 是一个全局授时器，它直接使用一个单点服务来分配时间。TSO 的方式很简单，但会有单点故障问题，单点也可能会有性能问题。

TiKV 采用了 TSO 的方式进行全局授时，主要是为了简单。至于单点故障问题，我们通过 Raft 做到了自动 fallover 处理。而对于单点性能问题，TiKV 主要针对的是 PB 以及 PB 以下级别的中小规模集群，所以在性能上面只要能保证每秒百万级别的时间分配就可以了，而网络延迟上面，TiKV 并没有全球跨 IDC 的需求，在单 IDC 或者同城 IDC 情况下，网络速度都很快，即使是异地 IDC，也因为有专线不会有太大的延迟。

解决了时间问题，下一个问题就是我们采用何种的分布式事务算法，最通常的就是使用 2 PC，但通常的 2 PC 算法在一些极端情况下面会有问题，所以业界要不通过 Paxos，要不就是使用 3 PC 等算法。在这里，TiKV 参考 Percolator，使用了另一种增强版的 2 PC 算法。

这里先简单介绍下 Percolator 的分布式事务算法，Percolator 使用了乐观锁，也就是会先缓存事务要修改的数据，然后在 Commit 提交的时候，对要更改的数据进行加锁处理，然后再更新。采用乐观锁的好处在于对于很多场景能提高整个系统的并发处理能力，但在冲突严重的情况下反而没有悲观锁高效。

对于要修改的一行数据，Percolator 会有三个字段与之对应，Lock，Write 和 Data：

+ Lock，就是要修改数据的实际 lock，在一个 Percolator 事务里面，有一个 primary key，还有其它 secondary keys， 只有 primary key 先加锁成功，我们才会再去尝试加锁后续的 secondary keys。

+ Write，保存的是数据实际提交写入的 commit timestamp，当一个事务提交成功之后，我们就会将对应的修改行的 commit timestamp 写入到 Write 上面。

+ Data，保存实际行的数据。

当事务开始的时候，我们会首先得到一个 start timestamp，然后再去获取要修改行的数据，在 Get 的时候，如果这行数据上面已经有 Lock 了，那么就可能终止当前事务，或者尝试清理 Lock。

当我们要提交事务的时候，先得到 commit timestamp，会有两个阶段：

1. Prewrite：先尝试给 primary key 加锁，然后尝试给 second keys 加锁。如果对应 key 上面已经有 Lock，或者在 start timestamp 之后，Write 上面已经有新的写入，Prewrite 就会失败，我们就会终止这次事务。在加锁的时候，我们也会顺带将数据写入到 Data 上面。

2. Commit：当所有涉及的数据都加锁成功之后，我们就可以提交 primay key，这时候会先判断之前加的 Lock 是否还在，如果还在，则删掉 Lock，将 commit timestamp 写入到 Write。当 primary key 提交成功之后，我们就可以异步提交 second keys，我们不用在乎 primary keys 是否能提交成功，即使失败了，也有机制能保证数据被正常提交。

在 TiKV 里面，事务的实现主要包括两块，一个是集成在 TiDB 中的 [tikv client](https://github.com/pingcap/tidb/tree/master/store/tikv)，而另一个则是在 TiKV 中的 [storage](https://github.com/pingcap/tikv/tree/master/src/storage) mod 里面，后面我们会详细的介绍。

## RPC 框架

RPC 应该是分布式系统里面常用的一种网络交互方式，但实现一个简单易用并且高效的 RPC 框架并不是一件容易的事情，幸运的是，现在有很多可以供我们进行选择。

TiKV 从最开始设计的时候，就希望使用 gRPC，但 Rust 当时并没有能在生产环境中可用的 gRPC 实现，我们只能先基于 mio 自己做了一个 RPC 框架，但随着业务的复杂，这套 RPC 框架开始不能满足需求，于是我们决定，直接使用 Rust 封装 Google 官方的 C gRPC，这样就有了 [grpc-rs](https://github.com/pingcap/grpc-rs)。

这里先说一下为什么我们决定使用 gRPC，主要有如下原因：

+ gRPC 应用广泛，很多知名的开源项目都使用了，譬如 Kubernetes，etcd 等。

+ gRPC 有多种语言支持，我们只要定义好协议，其他语言都能直接对接。

+ gRPC 有丰富的接口，譬如支持 unary，client streaming，server streaming 以及 duplex streaming。

+ gRPC 使用 protocol buffer，能高效的处理消息的编解码操作。

+ gRPC 基于 HTTP/2，一些 HTTP/2 的特性，譬如 duplexing，flow control 等。

最开始开发 rust gRPC 的时候，我们先准备尝试基于一个 rust 的版本来开发，但无奈遇到了太多的 panic，果断放弃，于是就将目光放到了 Google gRPC 官方的库上面。Google gRPC 库提供了多种语言支持，譬如 C++，C#，Python，这些语言都是基于一个核心的 C gRPC 来做的，所以我们自然选择在 Rust 里面直接使用 C gRPC。

因为 Google 的 C gRPC 是一个异步模型，为了简化在 rust 里面异步代码编写的难度，我们使用 rust Future 库将其重新包装，提供了 Future API，这样就能按照 Future 的方式简单使用了。

关于 gRPC 的详细介绍以及 rust gRPC 的设计还有使用，我们会在后面的文章中详细介绍。

## 监控

很难想象一个没有监控的分布式系统是如何能稳定运行的。如果我们只有一台机器，可能时不时看下这台机器上面的服务还在不在，CPU 有没有问题这些可能就够了，但如果我们有成百上千台机器，那么势必要依赖监控了。

TiKV 使用的是 Prometheus，一个非常强大的监控系统。Prometheus 主要有如下特性：

+ 基于时序的多维数据模型，对于一个 metric，我们可以用多种 tag 进行多维区分。

+ 自定义的报警机制。

+ 丰富的数据类型，提供了 Counter，Guage，Histogram 还有 Summary 支持。

+ 强大的查询语言支持。

+ 提供 pull 和 push 两种模式支持。

+ 支持服务的动态发现和静态配置。

+ 能跟 Grafana 深度整合。

因为 Prometheus 并没有 Rust 的客户端，于是我们开发了 [rust-prometheus](https://github.com/pingcap/rust-prometheus)。Rust Prometheus 在设计上面参考了 Go Prometehus 的 API，但我们只支持了 最常用的 Counter，Guage 和 Histogram，并没有实现 Summary。

后面，我们会详细介绍 Prometheus 的使用，以及不同的数据类型的使用场景等。

## 测试

要做好一个分布式的 Key-Value Store，测试是非常重要的一环。 只有经过了最严格的测试，我们才能有信心去保证整个系统是可以稳定运行的。

从最开始开发 TiKV 的时候，我们就将测试摆在了最重要的位置，除了常规的 unit test，我们还做了更多，譬如：

+ Stability test，我们专门写了一个 stability test，随机的干扰整个系统，同时运行我们的测试程序，看结果的正确性。

+ Jepsen，我们使用 Jepsen 来验证 TiKV 的线性一致性。

+ Namazu，我们使用 Namazu 来干扰文件系统以及 TiKV 线程调度。

+ Failpoint，我们在 TiKV 很多关键逻辑上面注入了 fail point，然后在外面去触发这些 fail，在验证即使出现了这些异常情况，数据仍然是正确的。

上面仅仅是我们的一些测试案例，当代码 merge 到 master 之后，我们的 CI 系统在构建好版本之后，就会触发所有的 test 执行，只有当所有的 test 都完全跑过，我们才会放出最新的版本。

在 Rust 这边，我们根据 FreeBSD 的 Failpoint 开发了 [fail-rs](https://github.com/pingcap/fail-rs)，并已经在 TiKV 的 Raft 中注入了很多 fail，后面还会在更多地方注入。我们也会基于 Rust 开发更多的 test 工具，用来测试整个系统。

## 小结

上面仅仅列出了我们用 Rust 开发 TiKV 的过程中，一些核心模块的设计思路。这篇文章只是一个简单的介绍，后面我们会针对每一个模块详细的进行说明。还有一些功能我们现在是没有做的，譬如 open tracing，这些后面都会慢慢开始完善。

我们的目标是通过 TiKV，在分布式系统领域，提供一套 Rust 解决方案，形成一个 Rust ecosystem。这个目标很远大，欢迎任何感兴趣的同学加入。
