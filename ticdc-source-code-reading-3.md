---
title: TiCDC 源码阅读（三）TiCDC 集群工作过程解析
author: ['金灵']
date: 2023-01-17
summary: 本文是 TiCDC 源码解读的第三篇，主要内容是讲述 TiCDC 集群的启动及基本工作过程。
tags: ["TiCDC"]
---

## 内容概要

[TiCDC](https://docs.pingcap.com/zh/tidb/dev/ticdc-overview) 是一款 TiDB 增量数据同步工具，通过拉取上游 TiKV 的数据变更日志，TiCDC 可以将数据解析为有序的行级变更数据输出到下游。

本文是 TiCDC 源码解读的第三篇，主要内容是讲述 TiCDC 集群的启动及基本工作过程，将从如下几个方面展开：

1. TiCDC Server 启动过程，以及 Server / Capture / Owner / Processor Manager 概念和关系
2. TiCDC Changefeed 创建过程
3. Etcd 在 TiCDC 集群中的作用
4. Owner 和 Processor Manager 概念介绍，以及 Owner 选举和切换过程
5. Etcd Worker 在 TiCDC 中的作用

## 启动 TiCDC Server

启动一个 TiCDC Server 时，使用的命令如下，需要传入当前上游 TiDB 集群的 PD 地址。

```bash
cdc server --pd=http://127.0.0.1:2379
```

它会启动一个 TiCDC Server 运行实例，并且向 PD 的 ETCD Server 写入 TiCDC 相关的元数据，具体的 Key 如下：

```plain text
/tidb/cdc/default/__cdc_meta__/capture/${capture_id}

/tidb/cdc/default/__cdc_meta__/owner/${session_id}

```

第一个 Key 是 Capture Key，用于注册一个 TiCDC Server 上运行的 Capture 信息，每次启动一个 Capture 时都会写入相应的 Key 和 Value。

第二个 Key 是 Campaign Key，每个 Capture 都会注册这样一个 Key 用于竞选 Owner。第一个写入 Owner Key 的 Capture 将成为 Owner 节点。

[Server 启动](https://github.com/pingcap/tiflow/blob/v6.4.0/pkg/cmd/server/server.go#L290)，经过了解析 Server 启动参数，验证参数合法性，然后创建并且运行 TiCDC Server。[Server 运行](https://github.com/pingcap/tiflow/blob/v6.4.0/cdc/server/server.go#L256)的过程中，会启动多个运行线程。首先启动一个 Http Server 线程，对外提供 Http OpenAPI 访问能力。其次，会创建一系列运行在 Server 级别的资源，主要作用是辅助 Capture 线程运行。最重要的是创建并且运行 Capture 线程，它是 TiCDC Server 运行的主要功能提供者。

![image.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_d8678008a7.png)

[Capture 运行](https://github.com/pingcap/tiflow/blob/v6.4.0/cdc/capture/capture.go#L292)时，首先会将自己的 Capture Information 投入到 ETCD 中。然后启动两个线程，一个运行 `ProcessorManager`，负责所有 Processor 的管理工作。另外一个运行 `campaignOwner`，其内部会负责竞选 Owner，以及运行 Owner 职责。如下所示，TiCDC Server 启动之后，会创建一个 Capture 线程，而 Capture 在运行过程中又会创建 ProcessorManager 和 Owner 两个线程，各自负责不同的工作任务。

![image (1).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_1_7bb32134ad.png)

## 创建 TiCDC Changefeed

创建 changefeed 时使用的命令如下：

```plain text
cdc changefeed create --server=http://127.0.0.1:8300 --sink-uri="blackhole://" --changefeed-id="blackhole-test"
```

其中的 server 参数标识了一个运行中的 TiCDC 节点，它记录了启动时候的 PD 地址。在创建 changefeed 时，server 会访问该 PD 内的 ETCD Server，写入一个 Changefeed 的元数据信息。
```plain
/tidb/cdc/default/default/changefeed/info/${changefeed_id}

/tidb/cdc/default/default/changefeed/status/${changefeed_id}
```

- 第一个 Key 标识了一个 Changefeed，包括该 Changefeed 的各种静态元数据信息，比如 `changefeed-id`，`sink-uri`，以及一些其他标识运行时是为的数据。
- 第二个 Key 标识了该 Changefeed 的运行时进度，主要是记录了 `Checkpoint` 和 `ResolvedTs` 的推进情况，会不断地周期性地更新。

## Etcd 的作用

![image (2).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_2_1f5f680540.png)

ETCD 在整个 TiCDC 集群中承担了非常重要的元数据存储功能，它记录了 Capture 和 Changefeed 等重要信息。同时通过不断记录并且更新 Changefeed 的 Checkpoint 和 ResolvedTs，保证 Changefeed 能够稳步向前推进工作。从上图中我们可以知道，Capture 在启动的时候，自行将自己的元数据信息写入到 ETCD 中，在此之后，Changefeed 的创建，暂停，删除等操作，都是经由已经启动的 TiCDC Owner 来执行的，后者负责更新 ETCD。

## Owner 选举和切换

一个 TiCDC 集群中可以存在着多个 TiCDC 节点，每个节点上都运行着一个 campaignOwner 线程，负责竞选 Owner，如果竞选成功，则履行 Owner 的工作职责。集群中只有一个节点会竞选成功，然后执行 Owner 的工作逻辑，其他节点上的该线程会阻塞在竞选 Owner 上。

![image (3).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_3_72d9a28b48.png)

TiCDC Owner 的选举过程是基于 [ETCD Election](https://etcd.io/docs/v3.3/dev-guide/api_concurrency_reference_v3/#service-election-etcdserverapiv3electionv3electionpbv3electionproto) 实现的。每个 Capture 在启动之后，会创建 [ETCD Session](https://github.com/etcd-io/etcd/blob/main/client/v3/concurrency/session.go)，然后使用该 Session，调用 [NewElection](https://github.com/etcd-io/etcd/blob/main/client/v3/concurrency/election.go#L44) 方法，创建到 Owner Key `/tidb/cdc/${ClusterID}/__cdc_meta/owner` 的竞选，然后调用 [Election.Campaign](https://github.com/etcd-io/etcd/blob/main/client/v3/concurrency/election.go#L69) 开始竞选。基本的相关代码过程如下：

```go
sess, err := concurrency.NewSession(etcdClient, ttl) // ttl is set to 10s
if err != nil {
    return err
}

election := concurrency.NewElection(sess, key) // key is `/tidb/cdc/${ClusterID}/__cdc_meta/owner`

if err := election.Campaign(ctx); err != nil {
    return err
}

...

```

感兴趣的读者 ，可以通过 [Capture.Run](https://github.com/pingcap/tiflow/blob/master/cdc/capture/capture.go#L278) 方法作为入口，浏览这部分代码流程，加深对该过程的理解。在真实的集群运行过程中，多个 TiCDC 节点先后上线，在不同的时刻开始竞选 Owner，第一个向 ETCD 中写入 Owner Key 的实例将成为 Owner。如下图所示，TiCDC-0 在 t=1 时刻写入 Owner Key，将会成为 Owner，它在后续运行过程中如果遇到故障辞去了自己的 Ownership，那么 TiCDC-1 将会成为新的 Owner 节点。老旧的 Owner 节点重新上线，调用 `Election.Campaign` 方法重新竞选 Owner，循环往复。

![image (4).png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/image_4_989a386b6e.png)

## EtcdWorker 模块

[EtcdWorker](https://github.com/pingcap/tiflow/blob/v6.4.0/pkg/orchestrator/etcd_worker.go) 是 TiCDC 内部一个非常重要的模块，它主要负责从 ETCD 中读取数据，映射到 TiCDC 内存中，然后驱动 Owner 和 ProcessorManager 运行。在具体的实现中，[EtcdWorker](https://etcd.io/docs/v3.2/learning/api/#watch-api) 通过调用 ETCD Watch 接口，周期性地获取到所有和 TiCDC 相关的 Key 的变化情况，然后映射到其自身维护的 `GlobalReactorState` 结构体中，其定义如下所示，其中记录了 Capture，Changefeed，Owner 等信息。

```go
type GlobalReactorState struct {
    ClusterID      string
    Owner          map[string]struct{}
    Captures       map[model.CaptureID]*model.CaptureInfo
    Upstreams      map[model.UpstreamID]*model.UpstreamInfo
    Changefeeds    map[model.ChangeFeedID]*ChangefeedReactorState
    
    ....
}
```

Owner 和 ProcessorManager 都是一个 [Reactor 接口](https://github.com/pingcap/tiflow/blob/master/pkg/orchestrator/interfaces.go#L24)的实现，二者都借助 `GlobalReactorState` 提供的信息来推进工作进度。具体地，[Owner](https://github.com/pingcap/tiflow/blob/master/cdc/owner/owner.go#L164) 通过轮询每一个记录在 `GlobalReactorState` 中的 Changefeed，让每一个 Changefeed 都能够被稳步推进同步状态。同时也负责诸如 Pause / Resume / Remove 等和 Changefeed 的运行状态相关的工作。[ProcessorManager](https://github.com/pingcap/tiflow/blob/master/cdc/processor/manager.go#L105) 则轮询每一个 Processor，让它们能够及时更新自身的运行状态。

## 总结

以上就是本文的全部内容。希望读者能够理解如下几个问题：

-  TiCDC Server 启动，创建 Changefeed 和 ETCD 的交互过程。
- EtcdWorker 如何读取 ETCD 数据并且驱动 Owner 和 Processor Manager 运行。
- TiCDC Owner 的竞选和切换过程。

下一次我们将向大家介绍 TiCDC Changefeed 内部的 Scheduler 模块的工作原理。
