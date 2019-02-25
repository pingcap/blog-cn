---
title: TiKV 功能介绍 - PD Scheduler
author: ['唐刘']
date: 2017-01-23
summary: 在前面的文章里面，我们介绍了 PD 一些常用功能，以及它是如何跟 TiKV 进行交互的，这里，我们重点来介绍一下 PD 是如何调度 TiKV 的。
tags: ['TiKV', 'PD', '集群调度']
---


在前面的文章里面，我们介绍了 PD 一些常用功能，以及它是如何跟 TiKV 进行交互的，这里，我们重点来介绍一下 PD 是如何调度 TiKV 的。

## 介绍

假设我们只有一个 TiKV，那么根本就无需调度了，因为数据只可能在这一台机器上面，client 也只可能跟这一个 TiKV 进行交互。但我们知道，在分布式存储领域，这样的情况不可能一直持续，因为数据量的增量一定会超过当前机器的物理存储极限，必然我们需要将一部分数据迁移到其他机器上面去。

在之前的文章里面，我们介绍过，TiKV 是通过 range 的方式将数据进行切分的。我们使用 Region 来表示一个数据 range，每个 Region 有多个副本 peer，通常为了安全，我们会使用至少三个副本。

最开始系统初始化的时候，我们只有一个 region，当数据量持续增大，超过了 Region 设置的最大 size（64MB） 阈值的时候，region 就会分裂，生成两个新的 region。region 是 PD 调度 TiKV 的基本单位。当我们新增加一个 TiKV 的时候，PD 就会将原来TiKV 里面的一些 Region 调度到这个新增的 TiKV 上面，这样就能保证整个数据均衡的分布在多个 TiKV 上面。因为一个 Region 通常是 64MB，其实将一个 Region 从一个 TiKV 移动到另一个 TiKV，数据量的变更其实不大，所以我们可以直接使用 Region 的数量来大概的做数据的平衡。譬如，现在假设有六个 TiKV，我们有一百个 region，每个 Region 三个副本 peer，总共三百个 Region peer，我们只要保证每个 TiKV 有五十个左右的 Region peer，就大概知道数据是平衡了。

上面我们只是介绍了数据的调度，但实际情况比这个要复杂很多，我们不光要考虑数据的均衡，也需要考虑计算的均衡，这样才能保证整个 TiKV 集群更好更快的对外提供服务。因为 TiKV 使用的是 Raft 分布式一致性算法，Raft 有一个强约束就是为了保证线性一致性，所有的读写都必须通过 leader 发起（后续我们会支持 follower read，能分担读压力）。假设现在有三个 TiKV，如果几乎所有的 leader 都集中在某一个 TiKV 上面，那么会造成这个 TiKV 成为性能瓶颈，最好的做法就是 leader 能够均衡在不同的 TiKV 上面，这样整个系统都能对外提供服务。

所以，在 PD，我们主要会对两种资源进行调度，存储 storage 以及计算 leader。

## 关键 Interface 和 Structure

为了满足不同的调度需求，PD 将调度相关的操作都抽象成了 interface，外面可以自由组合形成自己的调度方案。

### Scheduler

Scheduler 是用来调度资源的接口，定义如下：

```go
// Scheduler is an interface to schedule resources.
type Scheduler interface {
	GetName() string
	GetResourceKind() ResourceKind
	Schedule(cluster *clusterInfo) Operator
}
```

`GetName` 返回 Scheduler 名字，不同的 scheduler 不能重名。`GetResourceKind` 则是返回这个 Scheduler 要处理的资源类型，现阶段我们就两种，一个 leader，一个 storage。
`Scheduler` 则是进行实际的调度，它需要的参数就是整个集群的信息  ，在里面会生成实际的调度操作 Operator。

### Operator

前面我们说了，PD 对于 TiKV 调度的基本单位就是 region，所以 Scheduler 生成的 Operator 就是对一个 Region 进行调度。Operator 定义如下：

```go
// Operator is an interface to schedule region.
type Operator interface {
	GetRegionID() uint64
	GetResourceKind() ResourceKind
	Do(region *regionInfo) (*pdpb.RegionHeartbeatResponse, bool)
}
```

`GetRegionID` 得到需要调度的 Region ID，`GetResourceKind` 的含义跟 Scheduler 的一样。`Do` 则是对这个 Region 执行实际的操作，返回一个 RegionHeartbeatResponse。在之前的文章里面，我们说过，PD 对于 TiKV 的调度操作，都是在 TiKV Region heartbeat 命令里面返回给 TiKV，然后 TiKV 再去执行的。

多个 Operator 也可以组合成一个更上层的 Operator，但需要注意，这些 Operator 一定要有相同的 ResourceKind，也就是说，我们不能在一组 Operator 里面操作不同的 resource。

### Selector / Filter

假设我们要进行 storage 的调度，选择了一个 region，那么我们就需要做的是将 region 里面的一个副本 peer，迁移到另外的一个新的 TiKV 上面。所以我们在调度的时候，就需要选择一个合适的需要调度的 TiKV，也就是 source，然后就是一个合适的将被调度到的 TiKV，也就是 target。这个就是通过 Selector 来完成的。

```go
// Selector is an interface to select source and target store to schedule.
type Selector interface {
	SelectSource(stores []*storeInfo, filters ...Filter) *storeInfo
	SelectTarget(stores []*storeInfo, filters ...Filter) *storeInfo
}
```

Selector 的接口非常的简单，就是根据传入的 storeInfo 列表，以及一批 Filter，选择合适的 source 和 target，供 scheduler 实际去调度。

Filter 的定义如下：

```go
// Filter is an interface to filter source and target store.
type Filter interface {
	// Return true if the store should not be used as a source store.
	FilterSource(store *storeInfo) bool
	// Return true if the store should not be used as a target store.
	FilterTarget(store *storeInfo) bool
}
```

如果 Filter 的函数返回 true，就表明我们不能选择这个 store。

### Controller

通常，我们希望调度越来越快就好，但是实际情况，我们必须要保证调度不能影响现有的系统，不能造成现有系统出现太大的波动。

譬如，在做 storage 的调度的时候，PD 需要将 region 的某一个副本从一个 TiKV 迁移到另一个 TiKV，该 region 的 leader peer 会首先在目标 TiKV 上面添加一个新的 peer，这时候的操作是 leader 会生成当前 region 的 snapshot，然后发给 follower。Follower 收到 snapshot 之后，apply 到自己的状态机里面。同时，leader 会给原来要迁移的 peer 发送删除命令，该 follower 会在状态机里面清掉对应的数据。虽然一个 region 大概是 64MB，但过于频繁的一下子删除 64MB 数据，或者新增 64MB 数据，对于整个系统都是一个不小的负担。所以我们一定要控制整个调度的速度。

```go
// Controller is an interface to control the speed of different schedulers.
type Controller interface {
	Ctx() context.Context
	Stop()
	GetInterval() time.Duration
	AllowSchedule() bool
}
```

Controller 主要用来负责控制整个调度的速度，`GetInterval` 返回调度的间隔时间，当上一次调度之后，需要等待多久开始下一次的调度。`AllowSchedule` 则是表明是否允许调度。

## Coordinator

PD 使用 Coodinator 来管理所有的 Scheduler 以及 Controlller。

```go
// ScheduleController combines Scheduler with Controller.
type ScheduleController struct {
	Scheduler
	Controller
}
```

通常，对于调度，Scheduler 和 Controller 是同时存在的，所以在 Coordinator 里面会使用 ScheduleController 来统一进行管理。

Coordinator 在 region heartbeat 的时候，会看这个 region 是否需要调度，如果需要，则进行调度。

另外，在 Coordinator 里面，我们还有一个 replicaCheckController 定期检查 region 是否需要调度。因为 PD 知道整个集群的情况，所以 PD 就知道什么时候该进行调度。譬如，假设 PD 发现一个 TiKV 已经当掉，那么就会对在这个 TiKV 有副本的 region 生成调度 Operator，移除这个坏掉的副本，添加另一个好的副本，当 region heartbeat 上来的时候，直接接返回这个调度策略让 TiKV 去执行。

## 小结

这里简单的介绍了 PD 调度器的基本原理，需要调度的资源，以及一些关键的调度 Interface 以及 Structure，后面，我们会详细的介绍一些特定的调度策略，以及 PD 是如何通过 label 来进行更精确的调度的。
