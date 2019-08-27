---
title: TiDB Binlog 源码阅读系列文章（四）Pump server 介绍
author: ['satoru']
date: 2019-08-22
summary: 本文将继续介绍 Pump server 的实现，对应的源码主要集中在 TiDB Binlog 仓库的 pump/server.go 文件中。
tags: ['TiDB Binlog 源码阅读','社区']
---

在 [《TiDB Binlog 源码阅读系列文章（三）Pump client 介绍》](https://pingcap.com/blog-cn/tidb-binlog-source-code-reading-3/) 中，我们介绍了 TiDB 如何通过 Pump client 将 binlog 发往 Pump，本文将继续介绍 Pump server 的实现，对应的源码主要集中在 TiDB Binlog 仓库的 [`pump/server.go`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go) 文件中。

## 启动 Pump Server

Server 的启动主要由两个函数实现：[`NewServer`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L106) 和 [`(*Server).Start`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L317)。

`NewServer` 依照传入的配置项创建 Server 实例，初始化 Server 运行所必需的字段，以下简单说明部分重要字段：

1.  `metrics`：一个 [`MetricClient`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pkg/util/p8s.go#L36)，用于定时向 Prometheus Pushgateway 推送 metrics。

2.  `clusterID`：每个 TiDB 集群都有一个 ID，连接到同一个 TiDB 集群的服务可以通过这个 ID 识别其他服务是否属于同个集群。

3.  `pdCli`：[PD](https://github.com/pingcap/pd) Client，用于注册、发现服务，获取 Timestamp Oracle。

4.  `tiStore`：用于连接 TiDB storage engine，在这里主要用于查询事务相关的信息（可以通过 TiDB 中的对应 [interface 描述](https://github.com/pingcap/tidb/blob/v3.0.1/kv/kv.go#L259) 了解它的功能）。

5.  `storage`：Pump 的存储实现，从 TiDB 发过来的 binlog 就是通过它保存的，下一篇文章将会重点介绍。

Server 初始化以后，就可以用 `(*Server).Start` 启动服务。为了避免丢失 binlog，在开始对外提供 binlog 写入服务之前，[它会将当前 Server 注册到 PD 上，确保所有运行中的 Drainer 都已经观察到新增的 Pump 节点](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L323-L337)。这一步除了启动对外的服务，还开启了一些 Pump 正常运作所必须的辅助机制，下文会有更详细的介绍。

## Pump Server API

Pump Server 通过 gRPC 暴露出一些服务，这些接口定义在 [`tipb/pump.pb.go`](https://github.com/pingcap/tipb/blob/master/go-binlog/pump.pb.go#L312)，包含两个接口 `WriteBinlog`、 `PullBinlogs`。

### WriteBinlog

顾名思义，这是用于写入 binlog 的接口，上篇文章中 Pump client 调用的就是这个。客户端传入的请求，是以下的格式：

```go
type WriteBinlogReq struct {
  // The identifier of tidb-cluster, which is given at tidb startup.
  // Must specify the clusterID for each binlog to write.
  ClusterID uint64 `protobuf:"varint,1,opt,name=clusterID,proto3" json:"clusterID,omitempty"`
  // Payload bytes can be decoded back to binlog struct by the protobuf.
  Payload []byte `protobuf:"bytes,2,opt,name=payload,proto3" json:"payload,omitempty"`
}
```

其中 `Payload` 是一个用 `Protobuf` 序列化的 [binlog](https://github.com/pingcap/tipb/blob/master/go-binlog/binlog.pb.go#L223)，WriteBinlog 的 [主要流程](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L213-L227) 就是将请求中的 `Payload` 解析成 binlog 实例，然后调用 `storage.WriteBinlog` 保存下来。`storage.WriteBinlog` 将 binlog 持久化存储，并对 binlog 按 `start TS` / `commit TS` 进行排序，详细的实现将在下章展开讨论。

### PullBinlogs

PullBinlogs 是为 Drainer 提供的接口，用于按顺序获取 binlog。这是一个 streaming 接口，客户端请求后得到一个 stream，可以从中不断读取 binlog。请求的格式如下：

```go
type PullBinlogReq struct {
  // Specifies which clusterID of binlog to pull.
  ClusterID uint64 `protobuf:"varint,1,opt,name=clusterID,proto3" json:"clusterID,omitempty"`
  // The position from which the binlog will be sent.
  StartFrom Pos `protobuf:"bytes,2,opt,name=startFrom" json:"startFrom"`
}

// Binlogs are stored in a number of sequential files in a directory.
// The Pos describes the position of a binlog.
type Pos struct {
  // The suffix of binlog file, like .000001 .000002
  Suffix uint64 `protobuf:"varint,1,opt,name=suffix,proto3" json:"suffix,omitempty"`
  // The binlog offset in a file.
  Offset int64 `protobuf:"varint,2,opt,name=offset,proto3" json:"offset,omitempty"`
}

```

从名字可以看出，这个请求指定了 Drainer 要从什么时间点的 binlog 开始同步。虽然 Pos 中有 `Suffix` 和 `Offset` 两个字段，目前只有 `Offset` 字段是有效的，我们把它用作一个 `commit TS`，表示只拉取这个时间以后的 binlog。

PullBinlogs 的 [主要流程](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L275-L286)，是调用 `storage.PullCommitBinlogs` 得到一个可以获取序列化 binlog 的 channel，将这些 binlog 通过 `stream.Send` 接口逐个发送给客户端。

## 辅助机制

上文提到 Pump 的正常运作需要一些辅助机制，本节将逐一介绍这些机制。

### fake binlog

在 [《TiDB-Binlog 架构演进与实现原理》](https://pingcap.com/blog-cn/tidb-ecosystem-tools-1/) 一文中，对 fake binlog 机制有以下说明：

>“Pump 会定时（默认三秒）向本地存储中写入一条数据为空的 binlog，在生成该 binlog 前，会向 PD 中获取一个 tso，作为该 binlog 的 `start_ts` 与 `commit_ts`，这种 binlog 我们叫作 fake binlog。
>
>……Drainer 通过如上所示的方式对 binlog 进行归并排序，并推进同步的位置。那么可能会存在这种情况：某个 Pump 由于一些特殊的原因一直没有收到 binlog 数据，那么 Drainer 中的归并排序就无法继续下去，正如我们用两条腿走路，其中一只腿不动就不能继续前进。我们使用 Pump 一节中提到的 fake binlog 的机制来避免这种问题，Pump 每隔指定的时间就生成一条 fake binlog，即使某些 Pump 一直没有数据写入，也可以保证归并排序正常向前推进。”

[`genForwardBinlog`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L460) 实现了这个机制，它里面是一个定时循环，每隔一段时间（默认 3 秒，可通过 `gen-binlog-interval` 选项配置）检查一下是否有新的 binlog 写入，如果没有，就调用 `writeFakeBinlog` 写一条假的 binlog。

判断是否有新的 binlog 写入，是通过 `lastWriteBinlogUnixNano` 这个变量，每次有新的写入都会 [将这个变量设置为当前时间](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L193)。

### 垃圾回收

由于存储容量限制，显然 Pump 不能无限制地存储收到的 binlog，因此需要有一个 GC (Garbage Collection) 机制来清理没用的 binlog 释放空间，[`gcBinlogFile`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L527) 就负责 GC 的调度。有两个值会影响 GC 的调度：

1. `gcInterval`：控制 GC 检查的周期，目前写死在代码里的设置是 [1 小时](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L56)

2. `gcDuration`：binlog 的保存时长，每次 GC 检查就是 [通过当前时间和 `gcDuration` 计算出 GC 时间点](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L544-L545)，在这个时间点之前的 binlog 将被 GC 在 `gcBinlogFile` 的循环中，用 select 监控着 3 种情况：

```go
select {
case <-s.ctx.Done():
  log.Info("gcBinlogFile exit")
  return
case <-s.triggerGC:
  log.Info("trigger gc now")
case <-time.After(gcInterval):
}

```

3 个 case 分别对应：server 退出，外部触发 GC，定时检查这三种情况。其中 server 退出的情况我们直接退出循环。另外两种情况都会继续，计算 GC 时间点，交由 `storage.GC` 执行。

### Heartbeat

心跳机制用于定时（默认两秒）向 PD 发送 Server 最新状态，由 [`(*pumpNode).HeartBeat`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/node.go#L211) 实现。状态是由 JSON 编码的 [`Status`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pkg/node/node.go#L84) 实例，主要记录 `NodeID`、`MaxCommitTS` 之类的信息。

## HTTP API 实现

Pump Server 通过 HTTP 方式暴露出一些 API，主要提供运维相关的接口。

| 路径 | Handler | 说明 |
| :---------| :----------| :----------| 
| `GET /status` | [`Status`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L629) | 返回所有 Pump 节点的状态。 |
| `PUT /state/{nodeID}/{action}` | [`ApplyAction`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L722) | 支持 pause 和 close 两种 action，可以暂停和关闭 server。接到请求的 server 会确保用户指定的 nodeID 跟自己的 nodeID 相匹配，以防误操作。 |
| `GET /drainers` | [`AllDrainers`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L613) | 返回通过当前 PD 服务可以发现的所有 Drainer 的状态，一般用于调试时确定 Pump 是否能如预期地发现 Drainer。 |
| `GET /debug/binlog/{ts}` | [`BinlogByTS`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L644) | 通过指定的 timestamp 查询 binlog，如果查询结果是一条 Prewrite binlog，还会额外输出 MVCC 相关的信息。 |
| `POST /debug/gc/trigger` | [`TriggerGC`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L634) | 手动触发一次 GC，如果 GC 已经在运行中，请求将被忽略。 |

## 下线 Pump Server

下线一个 Pump server 的流程通常由 `binlogctl` 命令发起，例如：

```go
bin/binlogctl -pd-urls=localhost:2379 -cmd offline-pump -node-id=My-Host:8240
```

`binlogctl` 先通过 `nodeID` 在 PD 发现的 Pump 节点中找到指定的节点，然后调用上一小节中提到的接口 `PUT /state/{nodeID}/close`。

在 Server 端，`ApplyAction` 收到 close 后会将节点状态置为 Closing（Heartbeat 进程会定时将这类状态更新到 PD），然后另起一个 goroutine 调用 [`Close`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L834)。`Close` 首先调用 [`cancel`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L121)，通过 `context` 将关停信号发往协作的 goroutine，这些 goroutine 主要就是上文提到的辅助机制运行的 goroutine，例如在 `genForwardBinlog` 中设计了在 `context` 被 cancel 时退出：

```go
for {
  select {
  case <-s.ctx.Done():
     log.Info("genFakeBinlog exit")
     return
```

`Close` 用 `waitGroup` 等待这些 goroutine 全部退出。这时 Pump 仍然能正常提供 PullBinlogs 服务，但是写入功能 [已经停止](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L221)。`Close` 下一行调用了 `commitStatus`，这时节点的状态是 Closing，对应的分支调用了 [`waitSafeToOffline`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L769) 来确保到目前为止写入的 binlog 都已经被所有的 Drainer 读到了。`waitSafeToOffline` 先往 storage 中写入一条 fake binlog，由于此时写入功能已经停止，可以确定这将是这个 Pump 最后的一条 binlog。之后就是在循环中定时检查所有 Drainer 已经读到的 Binlog 时间信息，[直到这个时间已经大于 fake binlog 的 `CommitTS`](https://github.com/pingcap/tidb-binlog/blob/v3.0.1/pump/server.go#L795)。

`waitSafeToOffline` 等待结束后，就可以关停 gRPC 服务，释放其他资源。

## 小结

本文介绍了 Pump server 的启动、gRPC API 实现、辅助机制的设计以及下线服务的流程，希望能帮助大家在阅读源码时有一个更清晰的思路。在上面的介绍中，我们多次提到 `storage` 这个实体，用来存储和查询 binlog 的逻辑主要封装在这个模块内，这部分内容将在下篇文章为大家作详细介绍。
