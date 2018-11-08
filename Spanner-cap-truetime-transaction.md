---
title: Spanner - CAP, TrueTime and Transaction
author: ['唐刘']
date: 2017-02-21
summary: 最近大家非常关注的一件事情就是 Google Spanner Cloud 的发布，这应该算是 NewSQL 又一个里程碑的事件。在本篇文章中，唐刘同学与大家分享了他自己对 Spanner 的理解，Spanner 的一些关键技术的实现以及与 TiDB 的相关对比。
tags: ['Cloud', 'Spanner', 'CAP']
aliases:
  - /blog-cn/spanner-tangliu/
---


最近非常关注的一件事情就是 Google Spanner Cloud 的发布，这应该算是 NewSQL 又一个里程碑的事件。NewSQL 的概念应该就是在 12 年 Google Spanner 以及 F1 的论文发表之后，才开始慢慢流行，然后就开始有企业尝试根据 paper 做自己的 NewSQL，譬如国外的 CockroachDB 以及国内我们 PingCAP。

Spanner 的论文在很早就发布了，国内也有很多中文翻译，这里笔者只是想聊聊自己对 Spanner 的理解，以及 Spanner 的一些关键技术的实现，以及跟我们自己的 TiDB 的相关对比。

## CAP

在分布式领域，CAP 是一个完全绕不开的东西，大家应该早就非常熟悉，这里笔者只是简单的再次说明一下：

+ C：一致性，也就是通常说的线性一致性，假设在 T 时刻写入了一个值，那么在 T 之后的读取一定要能读到这个最新的值。
+ A：完全 100% 的可用性，也就是无论系统发生任何故障，都仍然能对外提供服务。
+ P：网络分区容忍性。

在分布式环境下面，P 是铁定存在的，也就是只要我们有多台机器，那么网络隔离分区就一定不可避免，所以在设计系统的时候我们就要选择到底是设计的是 AP 系统还是 CP 系统，但实际上，我们只要深入理解下 CAP，就会发现其实有时候系统设计上面没必要这么纠结，主要表现在：

1. 网络分区出现的概率很低，所以我们没必要去刻意去忽略 C 或者 A。多数时候，应该是一个 CA 系统。
2. CAP 里面的 A 是 100% 的可用性，但实际上，我们只需要提供 high availability，也就是仅仅需要满足 99.99% 或者 99.999% 等几个 9 就可以了。

Spanner 是一个 CP + HA 系统，官方文档说的可用性是优于 5 个 9 ，稍微小于 6 个 9，也就是说，Spanner 在系统出现了大的故障的情况下面，大概 31s+ 的时间就能够恢复对外提供服务，这个时间是非常短暂的，远远比很多外部的系统更加稳定。然后鉴于 Google 强大的自建网络，P 很少发生，所以 Spanner 可以算是一个 CA 系统。

TiDB 在设计的时候也是一个 CP + HA 系统，多数时候也是一个 CA 系统。如果出现了 P，也就是刚好对外服务的 leader 被隔离了，新 leader 大概需要 10s+ 以上的时间才能选举出来对外提供服务。当然，我们现在还不敢说系统的可用性在 6 个 9 了，6 个 9 现在还是我们正在努力的目标。

当然，无论是 Spanner 还是 TiDB，当整个集群真的出现了灾难性的事故，导致大多数节点都出现了问题，整个系统当然不可能服务了，当然这个概率是非常小的，我们可以通过增加更多的副本数来降低这个概率发生。据传在一些关键数据上面，Spanner 都有 7 个副本。

## TrueTime

最开始看到 Spanner 论文的时候，我是直接被 TrueTime 给惊艳到了，这特么的完全就是解决分布式系统时间问题的一个核弹呀（银弹我可不敢说）。

在分布式系统里面，时间到底有多么重要呢？之前笔者也写过一篇文章来聊过[分布式系统的时间](http://www.jianshu.com/p/8500882ab38c)问题，简单来说，我们需要有一套机制来保证相关事务之间的先后顺序，如果事务 T1 在事务 T2 开始之前已经提交，那么 T2 一定能看到 T1 提交的数据。

也就是事务需要有一个递增序列号，后开始的事务一定比前面开始的事务序列号要大。那么这跟时间又有啥关系呢，用一个全局序列号生成器不就行呢，为啥还要这么麻烦的搞一个 TrueTime 出来？笔者觉得有几个原因：

1. 全局序列号生成器是一个典型的单点，即使会做一些 failover 的处理，但它仍然是整个系统的一个瓶颈。同时也避免不了网络开销。但全局序列号的实现非常简单，Google 之前的 Percolator 以及现在 TiDB 都是采用这种方式。
2. 为什么要用时间？判断两个事件的先后顺序，时间是一个非常直观的度量方式，另外，如果用时间跟事件关联，那么我们就能知道某一个时间点整个系统的 snapshot。在 TiDB 的用户里面，一个非常典型的用法就是在游戏里面确认用户是否谎报因为回档丢失了数据，假设用户说在某个时间点得到某个装备，但后来又没有了，我们就可以直接在那个特定的时间点查询这个用户的数据，从而知道是否真的有问题。
3. 我们不光可以用时间来确定以前的 snapshot，同样也可以用时间来约定集群会在未来达到某个状态。这个典型的应用就是 schema change。虽然笔者不清楚 Spanner schema change 的实现，但 Google F1 有一篇 [Online, Asynchronous Schema Change in F1](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41376.pdf) 论文提到了相关的方法，而 TiDB 也是采用的这种实现方式。简单来说，对于一个 schema change，通常都会分几个阶段来完成，如果集群某个节点在未来一个约定的时间没达到这个状态，这个节点就需要自杀下线，防止因为数据不一致损坏数据。

使用 TrueTime，Spanner 可以非常方便的实现笔者提到的用法，但 TrueTime 也并不是万能的：

+ TrueTime 需要依赖 atomic clock 和 GPS，这属于硬件方案，而 Google 并没有论文说明如何构造 TrueTime，对于其他用户的实际并没有太多参考意义。
+ TrueTime 也会有误差范围，虽然非常的小，在毫秒级别以下，所以我们需要等待一个最大的误差时间，才能确保事务的相关顺序。

## Transaction

Spanner 默认将数据使用 range 的方式切分成不同的 splits，就跟 TiKV 里面 region 的概念比较类似。每一个 Split 都会有多个副本，分布在不同的 node 上面，各个副本之间使用 Paxos 协议保证数据的一致性。

Spanner 对外提供了 read-only transaction 和 read-write transaction 两种事务，这里简单的介绍一下，主要参考 Spanner 的[白皮书](https://cloud.google.com/spanner/docs/whitepapers/LifeCloudSpannerReadWrite.pdf)。

### Single Split Write

假设 client 要插入一行数据 Row 1，这种情况我们知道，这一行数据铁定属于一个 split，spanner 在这里使用了一个优化的 1PC 算法，流程如下：

1. API Layer 首先找到 Row 1 属于哪一个 split，譬如 Split 1。
2. API Layer 将写入请求发送给 Split 1 的 leader。
3. Leader 开始一个事务。
4. Leader 首先尝试对于 Row 1 获取一个 write lock，如果这时候有另外的 read-write transaction 已经对于这行数据上了一个 read lock，那么就会等待直到能获取到 write lock。
	+ 这里需要注意的是，假设事务 1 先 lock a，然后 lock b，而事务 2 是先 lock b，在 lock a，这样就会出现 dead lock 的情况。这里 Spanner 采用的是 `wound-wait` 的解决方式，新的事务会等待老的事务的 lock，而老的事务可能会直接 abort 掉新的事务已经占用的 lock。
5. 当 lock 被成功获取到之后，Leader 就使用 TrueTime 给当前事务绑定一个 timestamp。因为用 TrueTime，我们能够保证这个 timestamp 一定大于之前已经提交的事务 timestamp，也就是我们一定能够读取到之前已经更新的数据。
6. Leader 将这次事务和对应的 timestamp 复制给 Split 1 其他的副本，当大多数副本成功的将这个相关 Log 保存之后，我们就可以认为该事务已经提交（注意，这里还并没有将这次改动 apply）。
7. Leader 等待一段时间确保事务的 timestamp 有效（TrueTime 的误差限制），然后告诉 client 事务的结果。这个 `commit wait` 机制能够确保后面的 client 读请求一定能读到这次事务的改动。另外，因为 `commit wait` 在等待的时候，Leader 同时也在处理上面的步骤 6，等待副本的回应，这两个操作是并行的，所以 `commit wait` 开销很小。
8. Leader 告诉 client 事务已经被提交，同时也可以顺便返回这次事务的 timestamp。
9. 在 Leader 返回结果给 client 的时候，这次事务的改动也在并行的被 apply 到状态机里面。
	+ Leader 将事务的改动 apply 到状态机，并且释放 lock。
		- Leader 同时通知其他的副本也 apply 事务的改动。
		- 后续其他的事务需要等到这次事务的改动被 apply 之后，才能读取到数据。对于 read-write 事务，因为要拿 read lock，所以必须等到之前的 write lock 释放。而对于 read-only 事务，则需要比较 read-only 的 timestamp 是不是大于最后已经被成功 apply 的数据的 timestamp。

TiDB 现在并没有使用 1PC 的方式，但不排除未来也针对单个 region 的 read-write 事务，提供 1PC 的支持。

### Multi Split Write

上面介绍了单个 Split 的 write 事务的实现流程，如果一个 read-write 事务要操作多个 Split 了，那么我们就只能使用 2PC 了。

假设一个事务需要在 Split 1 读取数据 Row 1，同时将改动 Row 2，Row 3 分别写到 Split 2，Split 3，流程如下：

1. client 开始一个 read-write 事务。
2. client 需要读取 Row 1，告诉 API Layer 相关请求。
3. API Layer 发现 Row 1 在 Split 1。
4. API Layer 给 Split 1 的 Leader 发送一个 read request。
5. Split1 的 Leader 尝试将 Row 1 获取一个 read lock。如果这行数据之前有 write lock，则会持续等待。如果之前已经有另一个事务上了一个 read lock，则不会等待。至于 deadlock，仍然采用上面的  `wound-wait` 处理方式。
6. Leader 获取到 Row 1 的数据并且返回。
7. . Clients 开始发起一个 commit request，包括 Row 2，Row 3 的改动。所有的跟这个事务关联的 Split 都变成参与者 participants。
8.  一个 participant 成为协调者 coordinator，譬如这个 case 里面 Row 2 成为 coordinator。Coordinator 的作用是确保事务在所有 participants 上面要不提交成功，要不失败。这些都是在 participants 和 coordinator 各自的 Split Leader 上面完成的。
9.  Participants 开始获取 lock
	+ Split 2 对 Row 2 获取 write lock。
	+ Split 3 对 Row 3 获取 write lock。
	+ Split 1 确定仍然持有 Row 1 的 read lock。
	+ 每个 participant 的 Split Leader 将 lock 复制到其他 Split 副本，这样就能保证即使节点挂了，lock 也仍然能被持有。
	+ 如果所有的 participants 告诉 coordinator lock 已经被持有，那么就可以提交事务了。coordinator 会使用这个时候的时间点作为这次事务的提交时间点。
	+ 如果某一个 participant  告诉 lock 不能被获取，事务就被取消
10. 如果所有 participants 和 coordinator 成功的获取了 lock，Coordinator 决定提交这次事务，并使用 TrueTime 获取一个 timestamp。这个 commit 决定，以及 Split 2 自己的 Row 2 的数据，都会复制到 Split 2 的大多数节点上面，复制成功之后，就可以认为这个事务已经被提交。
11. Coordinator 将结果告诉其他的 participants，各个 participant 的 Leader 自己将改动复制到其他副本上面。
12. 如果事务已经提交，coordinator 和所有的 participants 就 apply 实际的改动。
13. Coordinator Leader 返回给 client 说事务已经提交成功，并且返回事务的 timestamp。当然为了保证数据的一致性，需要有 `commit-wait`。

 TiDB 也使用的是一个 2PC 方案，采用的是优化的类 Google Percolator 事务模型，没有中心的 coordinator，全部是靠 client 自己去协调调度的。另外，TiDB 也没有实现 `wound-wait`，而是对一个事务需要操作的 key 顺序排序，然后依次上 lock，来避免 deadlock。

### Strong Read

上面说了在一个或者多个 Split 上面 read-write 事务的处理流程，这里在说说 read-only 的事务处理，相比 read-write，read-only 要简单一点，这里以多个 Split 的 Strong read 为例。

假设我们要在 Split 1，Split 2 和 Split 3 上面读取 Row 1，Row 2 和 Row 3。

1. API Layer 发现 Row 1，Row 2，和 Row 3 在 Split1，Split 2 和 Split 3 上面。
2. API Layer  通过 TrueTime 获取一个 read timestamp（如果我们能够接受 Stale Read 也可以直接选择一个以前的 timestamp 去读）。
3. API Layer 将读的请求发给 Split 1，Split 2 和 Split 3 的一些副本上面，这里有几种情况：
	+ 多数情况下面，各个副本能通过内部状态和 TrueTime 知道自己有最新的数据，直接能提供 read。
	+ 如果一个副本不确定是否有最新的数据，就向 Leader 问一下最新提交的事务 timestamp 是啥，然后等到这个事务被 apply 了，就可以提供 read。
	+ 如果副本本来就是 Leader，因为 Leader 一定有最新的数据，所以直接提供 read。
4. 各个副本的结果汇总然会返回给 client。

当然，Spanner 对于 Read 还有一些优化，如果我们要进行 stale read，并且这个 stale 的时间在 10s 之前，那么就可以直接在任何副本上面读取，因为 Leader 会每隔 10s 将最新的 timestamp 更新到其他副本上面。

现在 TiDB 只能支持从 Leader 读取数据，还没有支持 follower read，这个功能已经实现，但还有一些优化需要进行，现阶段并没有发布。

TiDB 在 Leader 上面的读大部分走的是 lease read，也就是只要 Leader 能够确定自己仍然在 lease 有效范围里面，就可以直接读，如果不能确认，我们就会走 Raft 的 ReadIndex 机制，让 Leader 跟其他节点进行 heartbeat 交互，确认自己仍然是 Leader 之后再进行读操作。

## 小结

随着 Spanner Cloud 的发布，我们这边也会持续关注 Spanner  Cloud 的进展，TiDB 的原始模型就是基于 Spanner + F1 搭建起来，随着 Spanner Cloud 更多资料的公布，TiDB 也能有更多的参考。

另外，我们一直相信，我们走在正确的道路上面，如果你对我们的东西感兴趣，欢迎联系我。

+ 邮箱：tl@pingcap.com
+ 微信：siddontang
