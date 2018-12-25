---
title: TiDB 源码阅读系列文章（十八）tikv-client（上）
author: ['周昱行']
date: 2018-09-06
summary: 本文会详细介绍数据读写过程中 tikv-client 需要解决的几个具体问题，enjoy～
tags: ['源码阅读','TiDB','社区']
---


在整个 SQL 执行过程中，需要经过 Parser，Optimizer，Executor，DistSQL 这几个主要的步骤，最终数据的读写是通过 tikv-client 与 TiKV 集群通讯来完成的。

为了完成数据读写的任务，tikv-client 需要解决以下几个具体问题：

1. 如何定位到某一个 key 或 key range 所在的 TiKV 地址？

2. 如何建立和维护和 tikv-server 之间的连接？

3. 如何发送 RPC 请求？

4. 如何处理各种错误？

5. 如何实现分布式读取多个 TiKV 节点的数据？

6. 如何实现 2PC 事务？

我们接下来就对以上几个问题逐一解答，其中 5、6 会在下篇中介绍。


## 如何定位 key 所在的 tikv-server

我们需要回顾一下之前 [《三篇文章了解 TiDB 技术内幕——说存储》](https://pingcap.com/blog-cn/tidb-internal-1/) 这篇文章中介绍过的一个重要的概念：Region。

TiDB 的数据分布是以 Region 为单位的，一个 Region 包含了一个范围内的数据，通常是 96MB 的大小，Region 的 meta 信息包含了 StartKey 和 EndKey 这两个属性。当某个 key >= StartKey && key < EndKey 的时候，我们就知道了这个 key 所在的 Region，然后我们就可以通过查找该 Region 所在的 TiKV 地址，去这个地址读取这个 key 的数据。

获取 key 所在的 Region, 是通过向 PD 发送请求完成的。PD client 实现了这样一个接口：

[GetRegion(ctx context.Context, key []byte) (*metapb.Region, *metapb.Peer, error)](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/pd/pd-client/client.go#L49)

通过调用这个接口，我们就可以定位这个 key 所在的 Region 了。

如果需要获取一个范围内的多个 Region，我们会从这个范围的 StartKey 开始，多次调用 `GetRegion` 这个接口，每次返回的 Region 的 EndKey 做为下次请求的 StartKey，直到返回的 Region 的 EndKey 大于请求范围的 EndKey。

以上执行过程有一个很明显的问题，就是我们每次读取数据的时候，都需要先去访问 PD，这样会给 PD 带来巨大压力，同时影响请求的性能。

为了解决这个问题，tikv-client 实现了一个 [RegionCache](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L50)  的组件，缓存 Region 信息， 当需要定位 key 所在的 Region 的时候，如果 RegionCache 命中，就不需要访问 PD 了。RegionCache 的内部，有两种数据结构保存 Region 信息，一个是 [map](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L55)，另一个是 [b-tree](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L56)，用 map 可以快速根据 region ID 查找到 Region，用 b-tree 可以根据一个 key 找到包含该 key 的 Region。

严格来说，PD 上保存的 Region 信息，也是一层 cache，真正最新的 Region 信息是存储在 tikv-server 上的，每个 tikv-server 会自己决定什么时候进行 Region 分裂，在 Region 变化的时候，把信息上报给 PD，PD 用上报上来的 Region 信息，满足 tidb-server 的查询需求。

当我们从 cache 获取了 Region 信息，并发送请求以后， tikv-server 会对 Region 信息进行校验，确保请求的 Region 信息是正确的。

如果因为 Region 分裂，Region 迁移导致了 Region 信息变化，请求的 Region 信息就会过期，这时 tikv-server 就会返回 Region 错误。遇到了 Region 错误，我们就需要[清理 RegionCache](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L318)，重新[获取最新的 Region 信息](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L329)，并重新发送请求。


## 如何建立和维护和 tikv-server 之间的连接

当 TiDB 定位到 key 所在的 tikv-server 以后，就需要建立和 TiKV 之间的连接，我们都知道， TCP 连接的建立和关闭有不小的开销，同时会增大延迟，使用连接池可以节省这部分开销，TiDB 和 tikv-server 之间也维护了一个连接池 [connArray](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L83)。

TiDB 和 TiKV 之间通过 gRPC 通信，而 gPRC 支持在单 TCP 连接上多路复用，所以多个并发的请求可以在单个连接上执行而不会相互阻塞。

理论上一个 tidb-server 和一个 tikv-server 之间只需要维护一个连接，但是在性能测试的时候发现，单个连接在并发-高的时候，会成为性能瓶颈，所以实际实现的时候，tidb-server 对每一个 tikv-server 地址维护了多个连接，[并以 round-robin 算法选择连接](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L159)发送请求。连接的个数可以在 [config](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/config/config.toml.example#L215) 文件里配置，默认是 16。


## 如何发送 RPC 请求

tikv-client 通过 [tikvStore](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/kv.go#L127) 这个类型，实现 [kv.Storage](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/kv/kv.go#L247) 这个接口，我们可以把 tikvStore 理解成 tikv-client 的一个包装。外部调用 `kv.Storage` 的接口，并不需要关心 RPC 的细节，RPC 请求都是 tikvStore 为了实现 `kv.Storage` 接口而发起的。

实现不同的 `kv.Storage` 接口需要发送不同的 RPC 请求。比如实现 [Snapshot.BatchGet](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/kv/kv.go#L233) 需要[tikvpb.TikvClient.KvBatchGet](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L61) 方法；实现  [Transaction.Commit](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/kv/kv.go#L128)，需要 [tikvpb.TikvClient.KvPrewrite](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L57),  [tikvpb.TikvClient.KvCommit](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L58)  等多个方法。

在 tikvStore 的实现里，并没有直接调用 RPC 方法，而是通过一个 [Client](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L76) 接口调用，做这一层的抽象的主要目的是为了让下层可以有不同的实现。比如用来测试的 [mocktikv 就自己实现了 Client 接口](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/mockstore/mocktikv/rpc.go#L493)，通过本地调用实现，并不需要调用真正的 RPC。

[rpcClient](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L180) 是真正实现 RPC 请求的 Client 实现，通过调用 [tikvrpc.CallRPC](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/tikvrpc/tikvrpc.go#L419)，发送 RPC 请求。`tikvrpc.CallRPC` 再往下层走，就是调用具体[每个 RPC  生成的代码](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L152)了，到了生成的代码这一层，就已经是 gRPC 框架这一层的内容了，我们就不继续深入解析了，感兴趣的同学可以研究一下 gRPC 的实现。


## 如何处理各种错误

我们前面提到 RPC 请求都是通过 [Client](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L76) 接口发送的，但实际上这个接口并没有直接被各个 tikvStore 的各个方法调用，而是通过一个 [RegionRequestSender](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_request.go#L46) 的对象调用的。

`RegionRequestSender` 主要的工作除了发送 RPC 请求，还要负责处理各种可以重试的错误，比如网络错误和部分 Region 错误。

**RPC 请求遇到的错误主要分为两大类：Region 错误和网络错误。**

[Region  错误](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/tikvrpc/tikvrpc.go#L359) 是由 tikv-server 收到请求后，在 response 里返回的，常见的有以下几种:

1. [NotLeader](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/errorpb/errorpb.pb.go#L207)

    这种错误的原因通常是 Region 的调度，PD 为了负载均衡，可能会把一个热点 Region 的 leader 调度到空闲的 tikv-server 上，而请求只能由 leader 来处理。遇到这种错误就需要 tikv-client 重试，把请求发给新的 leader。

2. [StaleEpoch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/errorpb/errorpb.pb.go#L210)

    这种错误主要是因为 Region 的分裂，当 Region 内的数据量增多以后，会分裂成多个新的 Region。新的 Region 包含的 range  是不同的，如果直接执行，返回的结果有可能是错误的，所以 TiKV 就会拒绝这个请求。tikv-client 需要从 PD 获取最新的 Region 信息并重试。

3. [ServerIsBusy](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/errorpb/errorpb.pb.go#L211)

    这个错误通常是因为 tikv-server 积压了过多的请求处理不完，tikv-server 如果不拒绝这个请求，队列会越来越长，可能等到客户端超时了，请求还没有来的及处理。所以做为一种保护机制，tikv-server 提前返回错误，让客户端等待一段时间后再重试。

另一类错误是网络错误，错误是由 [SendRequest 的返回值](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_request.go#L129) 返回的 error 的，遇到这种错误通常意味着这个 tikv-server 没有正常返回请求，可能是网络隔离或 tikv-server down 了。tikv-client 遇到这种错误，会调用 [OnSendFail](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_request.go#L140) 方法，处理这个错误，会在 RegionCache 里把这个请求失败的 tikv-server 上的[所有 region 都 drop 掉](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L453)，避免其他请求遇到同样的错误。

当遇到可以重试的错误的时候，我们需要等待一段时间后重试，我们需要保证每次重试等待时间不能太短也不能太长，太短会造成多次无谓的请求，增加系统压力和开销，太长会增加请求的延迟。我们用指数退避的算法来计算每一次重试前的等待时间，这部分的逻辑是在 [Backoffer](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/backoff.go#L176) 里实现的。

在上层执行一个 SQL 语句的时候，在 tikv-client 这一层会触发多个顺序的或并发的请求，发向多个 tikv-server，为了保证上层 SQL  语句的超时时间，我们需要考虑的不仅仅是单个 RPC 请求，还需要考虑一个 query 整体的超时时间。

为了解决这个问题，`Backoffer` 实现了 [fork](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/backoff.go#L267) 功能， 在发送每一个子请求的时候，需要 fork 出一个 `child Backoffer`，`child Backoffer` 负责单个 RPC 请求的重试，它记录了 `parent Backoffer` 已经等待的时间，保证总的等待时间，不会超过 query 超时时间。

对于不同错误，需要等待的时间是不一样的，每个 `Backoffer` 在创建时，会[根据不同类型，创建不同的 backoff 函数](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/backoff.go#L96)。


**以上就是 tikv-client 上篇的内容，我们在下篇会详细介绍实现分布式计算相关的 [copIterator](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/coprocessor.go#L354) 和实现分布式事务的 [twoPCCommiter](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/2pc.go#L66)。**