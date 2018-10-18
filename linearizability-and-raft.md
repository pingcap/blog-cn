---
title: 线性一致性和 Raft
author: ['沈泰宁']
date: 2018-10-18
summary: 本篇文章会讨论一下线性一致性和 Raft，以及 TiKV 针对前者的一些优化。
tags: ['Raft','线性一致','TiKV']
---

在讨论分布式系统时，共识算法（Consensus algorithm）和一致性（Consistency）通常是讨论热点，两者的联系很微妙，很容易搞混。一些常见的误解：使用了 Raft [0] 或者 paxos 的系统都是线性一致的（Linearizability [1]，即强一致），其实不然，共识算法只能提供基础，要实现线性一致还需要在算法之上做出更多的努力。以 TiKV 为例，它的共识算法是 Raft，在 Raft 的保证下，TiKV 提供了满足线性一致性的服务。

本篇文章会讨论一下线性一致性和 Raft，以及 TiKV 针对前者的一些优化。

## 线性一致性

什么是一致性，简单的来说就是评判一个并发系统正确与否的标准。线性一致性是其中一种，CAP [2] 中的 C 一般就指它。什么是线性一致性，或者说怎样才能达到线性一致？在回答这个问题之前先了解一些背景知识。

### 背景知识

为了回答上面的问题，我们需要一种表示方法描述分布式系统的行为。分布式系统可以抽象成几个部分:

+ Client
+ Server
+ Events
    - Invocation
    - Response
+ Operations
    - Read
    - Write

一个分布式系统通常有两种角色，Client 和 Server。Client 通过发起请求来获取 Server 的服务。一次完整请求由两个事件组成，Invocation（以下简称 Inv）和 Response（以下简称 Resp）。一个请求中包含一个 Operation，有两种类型 Read 和 Write，最终会在 Server 上执行。

说了一堆不明所以的概念，现在来看如何用这些表示分布式系统的行为。

![](http://upload-images.jianshu.io/upload_images/542677-4c8d7e86005ea29d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图展示了 Client A 的一个请求从发起到结束的过程。变量 x 的初始值是 1，“x R() A” 是一个事件 Inv 意思是 A 发起了读请求，相应的 “x OK(1) A” 就是事件 Resp，意思是 A 读到了 x 且值为 1，Server 执行读操作（Operation）。

### 如何达到线性一致

背景知识介绍完了，怎样才能达到线性一致？这就要求 Server 在执行 Operations 时需要满足以下三点：

1. 瞬间完成（或者原子性）

2. 发生在 Inv 和 Resp 两个事件之间

3. 反映出“最新”的值

下面我举一个例子，用以解释上面三点。

**例：**

![](http://upload-images.jianshu.io/upload_images/542677-c45879eb3d1f819f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先下结论，上图表示的行为满足线性一致。

对于同一个对象 x，其初始值为 1，客户端 ABCD 并发地进行了请求，按照真实时间（real-time）顺序，各个事件的发生顺序如上图所示。对于任意一次请求都需要一段时间才能完成，例如 A，“x R() A” 到 “x Ok(1) A” 之间的那条线段就代表那次请求花费的时间，而请求中的读操作在 Server 上的执行时间是很短的，相对于整个请求可以认为瞬间，读操作表示为点，并且在该线段上。线性一致性中没有规定读操作发生的时刻，也就说该点可以在线段上的任意位置，可以在中点，也可以在最后，当然在最开始也无妨。

第一点和第二点解释的差不多了，下面说第三点。

反映出“最新”的值？我觉得三点中最难理解就是它了。先不急于对“最新”下定义，来看看上图中 x 所有可能的值，显然只有 1 和 2。四个次请求中只有 B 进行了写请求，改变了 x 的值，我们从 B 着手分析，明确 x 在各个时刻的值。由于不能确定 B 的 W（写操作）在哪个时刻发生，能确定的只有一个区间，因此可以引入**上下限**的概念。对于 x=1，它的上下限为**开始到事件“x W(2) B”**，在这个范围内所有的读操作必定读到 1。对于 x=2，它的上下限为 **事件“x Ok() B”** 到结束，在这个范围内所有的读操作必定读到 2。那么“x W(2) B”到“x Ok() B”这段范围，x 的值是什么？**1 或者 2**。由此可以将 x 分为三个阶段，各阶段"最新"的值如下图所示:

![](http://upload-images.jianshu.io/upload_images/542677-f0ca7b955e87706c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

清楚了 x 的变化后理解例子中 A C D 的读到结果就很容易了。

最后返回的 D 读到了 1，看起来是 “stale read”，其实并不是，它仍满足线性一致性。D 请求横跨了三个阶段，而读可能发生在任意时刻，所以 1 或 2 都行。同理，A 读到的值也可以是 2。C 就不太一样了，C 只有读到了 2 才能满足线性一致。因为 “x R() C” 发生在 “x Ok() B” 之后（happen before [3]），可以推出 R 发生在 W 之后，那么 R 一定得读到 W 完成之后的结果：2。

**一句话概括：在分布式系统上实现寄存器语义。**

## 实现线性一致

如开头所说，一个分布式系统正确实现了共识算法并不意味着能线性一致。共识算法只能保证多个节点对某个对象的状态是一致的，以 Raft 为例，它只能保证不同节点对 Raft Log（以下简称 Log）能达成一致。那么 Log 后面的状态机（state machine）的一致性呢？并没有做详细规定，用户可以自由实现。

### Raft

Raft 是一个强 Leader 的共识算法，只有 Leader 能处理客户端的请求，集群的数据（Log）的流向是从 Leader 流向 Follower。其他的细节在这就不赘述了，网上有很多资料 [4]。

#### In Practice

以 TiKV 为例，TiKV 内部可分成多个模块，Raft 模块，RocksDB 模块，两者通过 Log 进行交互，整体架构如下图所示，consensus 就是 Raft 模块，state machine 就是 RocksDB 模块。

![](http://upload-images.jianshu.io/upload_images/542677-c0b61a9d5b8f6a4a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Client 将请求发送到 Leader 后，Leader 将请求作为一个 Proposal 通过 Raft 复制到自身以及 Follower 的 Log 中，然后将其 commit。TiKV 将 commit 的 Log 应用到 RocksDB 上，由于 Input（即 Log）都一样，可推出各个 TiKV 的状态机（即 RocksDB）的状态能达成一致。但实际多个 TiKV 不能保证同时将某一个 Log 应用到 RocksDB 上，也就是说各个节点不能**实时**一致，加之 Leader 会在不同节点之间切换，所以 Leader 的状态机也不总有最新的状态。Leader 处理请求时稍有不慎，没有在最新的状态上进行，这会导致整个系统违反线性一致性。**好在有一个很简单的解决方法：依次应用 Log，将应用后的结果返回给 Client。**

这方法不仅简单还通用，读写请求都可以这样实现。这个方法依据 commit index 对所有请求都做了排序，使得每个请求都能反映出状态机在执行完前一请求后的状态，可以认为 commit 决定了 R/W 事件发生的顺序。Log 是严格全序的（total order），那么自然所有 R/W 也是全序的，将这些 R/W 操作一个一个应用到状态机，所得的结果必定符合线性一致性。这个方法的缺点很明显，性能差，因为所有请求在 Log 那边就被序列化了，无法并发的操作状态机。

这样的读简称 LogRead。由于读请求不改变状态机，这个实现就显得有些“重“，不仅有 RPC 开销，还有写 Log 开销。优化的方法大致有两种：

* ReadIndex

* LeaseRead

### ReadIndex

相比于 LogRead，ReadIndex 跳过了 Log，节省了磁盘开销，它能大幅提升读的吞吐，减小延时（但不显著）。Leader 执行 ReadIndex 大致的流程如下：

1. 记录当前的 commit index，称为 ReadIndex

2. 向 Follower 发起一次心跳，如果大多数节点回复了，那就能确定现在仍然是 Leader

3. 等待状态机**至少**应用到 ReadIndex 记录的 Log

4. 执行读请求，将结果返回给 Client

第 3 点中的“至少”是关键要求，它表明状态机应用到 ReadIndex 之后的状态都能使这个请求满足线性一致，不管过了多久，也不管 Leader 有没有飘走。为什么在 ReadIndex 只有就满足了线性一致性呢？之前 LogRead 的读发生点是 commit index，这个点能使 LogRead 满足线性一致，那显然发生这个点之后的 ReadIndex 也能满足。

### LeaseRead

LeaseRead 与 ReadIndex 类似，但更进一步，不仅省去了 Log，还省去了网络交互。它可以大幅提升读的吞吐也能显著降低延时。基本的思路是 Leader 取一个比 Election Timeout 小的租期，在租期不会发生选举，确保 Leader 不会变，所以可以跳过 ReadIndex 的第二步，也就降低了延时。 LeaseRead 的正确性和时间挂钩，因此时间的实现至关重要，如果漂移严重，这套机制就会有问题。

#### Wait Free

到此为止 Lease 省去了 ReadIndex 的第二步，实际能再进一步，省去第 3 步。这样的 LeaseRead 在收到请求后会立刻进行读请求，不取 commit index 也不等状态机。由于 Raft 的强 Leader 特性，在租期内的 Client 收到的 Resp 由 Leader 的状态机产生，所以只要状态机满足线性一致，那么在 Lease 内，不管何时发生读都能满足线性一致性。有一点需要注意，只有在 Leader 的状态机应用了当前 term 的第一个 Log 后才能进行 LeaseRead。因为新选举产生的 Leader，它虽然有全部 committed Log，但它的状态机可能落后于之前的 Leader，状态机应用到当前 term 的 Log 就保证了新 Leader 的状态机一定新于旧 Leader，之后肯定不会出现 stale read。

## 写在最后

本文粗略地聊了聊线性一致性，以及 TiKV 内部的一些优化。最后留四个问题以便更好地理解本文：

1. 对于线性一致中的例子，如果 A 读到了 2，那么 x 的各个阶段是怎样的呢？

2. 对于下图，它符合线性一致吗？（温馨提示：请使用游标卡尺。;-P）

    ![](https://upload-images.jianshu.io/upload_images/542677-0aec44f67708c1ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.  Leader 的状态机在什么时候没有最新状态？要线性一致性，Raft 该如何解决这问题？

4.  FollowerRead 可以由 ReadIndex 实现，那么能由 LeaseRead 实现吗？

如有疑问或想交流，欢迎联系我：shentaining@pingcap.com

[0].Ongaro, Diego. Consensus: Bridging theory and practice. Diss. Stanford University, 2014.

[1].Herlihy, Maurice P., and Jeannette M. Wing. "Linearizability: A correctness condition for concurrent objects." ACM Transactions on Programming Languages and Systems (TOPLAS) 12.3 (1990): 463-492.

[2].Gilbert, Seth, and Nancy Lynch. "Brewer's conjecture and the feasibility of consistent, available, partition-tolerant web services." Acm Sigact News 33.2 (2002): 51-59.

[3].Lamport, Leslie. "Time, clocks, and the ordering of events in a distributed system." Communications of the ACM 21.7 (1978): 558-565.

[4].[https://raft.github.io/](https://raft.github.io/)

