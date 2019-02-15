---
title: TiKV 功能介绍 - Raft 的优化
author: ['唐刘']
date: 2017-03-07
summary: 在分布式领域，为了保证数据的一致性，通常都会使用 Paxos 或者 Raft 来实现。但 Paxos 以其复杂难懂著称，相反 Raft 则是非常简单易懂，所以现在很多新兴的数据库都采用 Raft 作为其底层一致性算法，包括我们的 TiKV。
tags: ['TiKV', 'Raft', '性能优化']
---


在分布式领域，为了保证数据的一致性，通常都会使用 Paxos 或者 Raft 来实现。但 Paxos 以其复杂难懂著称，相反 Raft 则是非常简单易懂，所以现在很多新兴的数据库都采用 Raft 作为其底层一致性算法，包括我们的 TiKV。

当然，Raft 虽然简单，但如果单纯的按照 Paper 的方式去实现，性能是不够的。所以还需要做很多的优化措施。本文假定用户已经熟悉并了解过 Raft 算法，所以对 Raft 不会做过多说明。

## Simple Request Flow

这里首先介绍一下一次简单的 Raft 流程：

1. Leader 收到 client 发送的 request。
2. Leader 将 request append 到自己的 log。
3. Leader 将对应的 log entry 发送给其他的 follower。
4. Leader 等待 follower 的结果，如果大多数节点提交了这个 log，则 apply。
5. Leader 将结果返回给 client。
6. Leader 继续处理下一次 request。

可以看到，上面的流程是一个典型的顺序操作，如果真的按照这样的方式来写，那性能是完全不行的。

## Batch and Pipeline

首先可以做的就是 batch，大家知道，在很多情况下面，使用 batch 能明显提升性能，譬如对于 RocksDB 的写入来说，我们通常不会每次写入一个值，而是会用一个 WriteBatch 缓存一批修改，然后在整个写入。 对于 Raft 来说，Leader 可以一次收集多个 requests，然后一批发送给 Follower。当然，我们也需要有一个最大发送 size 来限制每次最多可以发送多少数据。

如果只是用 batch，Leader  还是需要等待 Follower 返回才能继续后面的流程，我们这里还可以使用 Pipeline 来进行加速。大家知道，Leader 会维护一个 NextIndex 的变量来表示下一个给 Follower 发送的 log 位置，通常情况下面，只要 Leader 跟 Follower 建立起了连接，我们都会认为网络是稳定互通的。所以当 Leader 给 Follower 发送了一批 log 之后，它可以直接更新 NextIndex，并且立刻发送后面的 log，不需要等待 Follower 的返回。如果网络出现了错误，或者 Follower 返回一些错误，Leader 就需要重新调整 NextIndex，然后重新发送 log 了。

## Append Log Parallelly

对于上面提到的一次 request 简易 Raft 流程来说，我们可以将 2 和 3 并行处理，也就是 Leader 可以先并行的将 log 发送给 Followers，然后再将 log append。为什么可以这么做，主要是因为在 Raft 里面，如果一个 log 被大多数的节点append，我们就可以认为这个 log 是被 committed 了，所以即使 Leader 再给 Follower 发送 log 之后，自己 append log 失败 panic 了，只要 `N / 2 + 1` 个 Follower 能接收到这个 log 并成功 append，我们仍然可以认为这个 log 是被 committed 了，被 committed 的 log 后续就一定能被成功 apply。

那为什么我们要这么做呢？主要是因为 append log 会涉及到落盘，有开销，所以我们完全可以在 Leader 落盘的同时让 Follower 也尽快的收到 log 并 append。

这里我们还需要注意，虽然 Leader 能在 append log 之前给 Follower 发 log，但是 Follower 却不能在 append log 之前告诉 Leader 已经成功 append 这个 log。如果 Follower 提前告诉 Leader 说已经成功 append，但实际后面 append log 的时候失败了，Leader 仍然会认为这个 log 是被 committed 了，这样系统就有丢失数据的风险了。

## Asynchronous Apply

上面提到，当一个 log 被大部分节点 append 之后，我们就可以认为这个 log 被 committed 了，被 committed 的 log 在什么时候被 apply 都不会再影响数据的一致性。所以当一个 log 被 committed 之后，我们可以用另一个线程去异步的 apply 这个 log。

所以整个 Raft 流程就可以变成：

1. Leader 接受一个 client 发送的 request。
2. Leader 将对应的 log 发送给其他 follower 并本地 append。
3. Leader 继续接受其他 client 的 requests，持续进行步骤 2。
4. Leader 发现 log 已经被 committed，在另一个线程 apply。
5. Leader 异步 apply log 之后，返回结果给对应的 client。

使用 asychronous apply 的好处在于我们现在可以完全的并行处理 append log 和 apply log，虽然对于一个 client 来说，它的一次 request 仍然要走完完整的 Raft 流程，但对于多个 clients 来说，整体的并发和吞吐量是上去了。

##  Now Doing…

### SST Snapshot

在 Raft 里面，如果 Follower 落后 Leader 太多，Leader 就可能会给 Follower 直接发送 snapshot。在 TiKV，PD 也有时候会直接将一个 Raft Group 里面的一些副本调度到其他机器上面。上面这些都会涉及到 Snapshot 的处理。

在现在的实现中，一个 Snapshot 流程是这样的：

1. Leader scan 一个 region 的所有数据，生成一个 snapshot file
2. Leader 发送 snapshot file 给 Follower
3. Follower 接受到 snapshot file，读取，并且分批次的写入到 RocksDB

如果一个节点上面同时有多个 Raft Group 的 Follower 在处理 snapshot file，RocksDB 的写入压力会非常的大，然后极易引起 RocksDB 因为 compaction 处理不过来导致的整体写入 slow 或者 stall。

幸运的是，RocksDB 提供了 [SST](https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files) 机制，我们可以直接生成一个 SST 的 snapshot file，然后 Follower 通过 injest 接口直接将 SST file load 进入 RocksDB。

### Asynchronous  Lease Read

在之前的 [Lease Read](http://mp.weixin.qq.com/s?__biz=MzI3NDIxNTQyOQ==&mid=2247484499&idx=1&sn=79acb9b4b2f8baa3296f2288c4a0a45b&scene=0#wechat_redirect) 文章中，我提到过 TiKV 使用 ReadIndex 和 Lease Read 优化了 Raft Read 操作，但这两个操作现在仍然是在 Raft 自己线程里面处理的，也就是跟 Raft 的 append log 流程在一个线程。无论 append log 写入 RocksDB 有多么的快，这个流程仍然会 delay Lease Read 操作。

所以现阶段我们正在做的一个比较大的优化就是在另一个线程异步实现 Lease Read。也就是我们会将 Leader Lease 的判断移到另一个线程异步进行，Raft 这边的线程会定期的通过消息去更新 Lease，这样我们就能保证 Raft 的 write 流程不会影响到 read。

## 总结

虽然外面有声音说 Raft 性能不好，但既然我们选择了 Raft，所以就需要对它持续的进行优化。而且现阶段看起来，成果还是很不错的。相比于 RC1，最近发布的 RC2 无论在读写性能上面，性能都有了极大的提升。但我们知道，后面还有很多困难和挑战在等着我们，同时我们也急需在性能优化上面有经验的大牛过来帮助我们一起改进，如果你对我们做的东西感兴趣，想让 Raft 快的飞起，欢迎联系我：

+ 邮箱：tl@pingcap.com
+ 微信：siddontang
