---
title: TiKV 功能介绍 - Lease Read
author: ['唐刘']
date: 2017-02-21
summary: 在 TiKV 里面，从最开始的 Raft log read，到后面的 Lease Read，我们一步一步的在保证线性一致性的情况下面改进着性能。后面，我们会引入更多的一致性测试 case 来验证整个系统的安全性，当然，也会持续的提升性能。
tags: ['TiKV', 'Raft', 'Lease Read','性能优化']
---


##  Raft log read

TiKV 是一个要保证线性一致性的分布式 KV 系统，所谓线性一致性，一个简单的例子就是在 t1 的时间我们写入了一个值，那么在 t1 之后，我们的读一定能读到这个值，不可能读到 t1 之前的值。

因为 Raft 本来就是一个为了实现分布式环境下面线性一致性的算法，所以我们可以通过 Raft 非常方便的实现线性 read，也就是将任何的读请求走一次 Raft log，等这个 log 提交之后，在 apply 的时候从状态机里面读取值，我们就一定能够保证这个读取到的值是满足线性要求的。

当然，大家知道，因为每次 read 都需要走 Raft 流程，所以性能是非常的低效的，所以大家通常都不会使用。

我们知道，在 Raft 里面，节点有三个状态，leader，candidate 和 follower，任何 Raft 的写入操作都必须经过 leader，只有 leader 将对应的 raft log 复制到 majority 的节点上面，我们才会认为这一次写入是成功的。所以我们可以认为，如果当前 leader 能确定一定是 leader，那么我们就可以直接在这个 leader 上面读取数据，因为对于 leader 来说，如果确认一个 log 已经提交到了大多数节点，在 t1 的时候 apply 写入到状态机，那么在 t1 之后后面的 read 就一定能读取到这个新写入的数据。

那么如何确认 leader 在处理这次 read 的时候一定是 leader 呢？在 Raft 论文里面，提到了两种方法。

## ReadIndex Read

第一种就是 ReadIndex，当 leader 要处理一个读请求的时候：

1. 将当前自己的 commit index 记录到一个 local 变量 ReadIndex 里面。
2. 向其他节点发起一次 heartbeat，如果大多数节点返回了对应的 heartbeat response，那么 leader 就能够确定现在自己仍然是 leader。
3. Leader 等待自己的状态机执行，直到 apply index 超过了 ReadIndex，这样就能够安全的提供 linearizable read 了。
4. Leader 执行 read 请求，将结果返回给 client。

可以看到，不同于最开始的通过 Raft log 的 read，ReadIndex read 使用了 heartbeat 的方式来让 leader 确认自己是 leader，省去了 Raft log 那一套流程。虽然仍然会有网络开销，但 heartbeat 本来就很小，所以性能还是非常好的。

但这里，需要注意，实现 ReadIndex 的时候有一个 corner case，在 etcd 和 TiKV 最初实现的时候，我们都没有注意到。也就是 leader 刚通过选举成为 leader 的时候，这时候的 commit index 并不能够保证是当前整个系统最新的 commit index，所以 Raft 要求当 leader 选举成功之后，首先提交一个 no-op 的 entry，保证 leader 的 commit index 成为最新的。

所以，如果在 no-op 的 entry 还没提交成功之前，leader 是不能够处理 ReadIndex 的。但之前 etcd 和 TiKV 的实现都没有注意到这个情况，也就是有 bug。解决的方法也很简单，因为 leader 在选举成功之后，term 一定会增加，在处理 ReadIndex 的时候，如果当前最新的 commit log 的 term 还没到新的 term，就会一直等待跟新的 term 一致，也就是 no-op entry 提交之后，才可以对外处理 ReadIndex。

使用 ReadIndex，我们也可以非常方便的提供 follower read 的功能，follower 收到 read 请求之后，直接给 leader 发送一个获取 ReadIndex 的命令，leader 仍然走一遍之前的流程，然后将 ReadIndex 返回给 follower，follower 等到当前的状态机的 apply index 超过 ReadIndex 之后，就可以 read 然后将结果返回给 client 了。

## Lease Read

虽然 ReadIndex 比原来的 Raft log read 快了很多，但毕竟还是有 Heartbeat 的开销，所以我们可以考虑做更进一步的优化。

在 Raft 论文里面，提到了一种通过 clock + heartbeat 的 lease read 优化方法。也就是 leader 发送 heartbeat 的时候，会首先记录一个时间点 start，当系统大部分节点都回复了 heartbeat response，那么我们就可以认为 leader  的 lease 有效期可以到 `start + election timeout / clock drift bound` 这个时间点。

为什么能够这么认为呢？主要是在于 Raft 的选举机制，因为 follower 会在至少 election timeout 的时间之后，才会重新发生选举，所以下一个 leader 选出来的时间一定可以保证大于 `start + election timeout / clock drift bound`。

虽然采用 lease 的做法很高效，但仍然会面临风险问题，也就是我们有了一个预设的前提，各个服务器的 CPU clock 的时间是准的，即使有误差，也会在一个非常小的 bound 范围里面，如果各个服务器之间 clock 走的频率不一样，有些太快，有些太慢，这套 lease 机制就可能出问题。

TiKV 使用了 lease read 机制，主要是我们觉得在大多数情况下面 CPU 时钟都是正确的，当然这里会有隐患，所以我们也仍然提供了 ReadIndex 的方案。

TiKV 的 lease read 实现在原理上面跟 Raft 论文上面的一样，但实现细节上面有些差别，我们并没有通过 heartbeat 来更新 lease，而是通过写操作。对于任何的写入操作，都会走一次 Raft log，所以我们在 propose 这次 write 请求的时候，记录下当前的时间戳 start，然后等到对应的请求 apply 之后，我们就可以续约 leader 的 lease。当然实际实现还有很多细节需要考虑的，譬如：

+ 我们使用的 monotonic raw clock，而 不是 monotonic clock，因为 monotonic clock 虽然不会出现 time jump back 的情况，但它的速率仍然会受到 NTP 等的影响。
+ 我们默认的 election timeout 是 10s，而我们会用 9s 的一个固定 max time 值来续约 lease，这样一个是为了处理 clock drift bound 的问题，而另一个则是为了保证在滚动升级 TiKV 的时候，如果用户调整了 election timeout，lease read 仍然是正确的。因为有了 max lease time，用户的 election timeout 只能设置的比这个值大，也就是 election timeout 只能调大，这样的好处在于滚动升级的时候即使出现了 leader 脑裂，我们也一定能够保证下一个 leader 选举出来的时候，老的 leader lease 已经过期了。

当然，使用 Raft log 来更新 lease 还有一个问题，就是如果用户长时间没有写入操作，这时候来的读取操作因为早就已经没有 lease 了，所以只能强制走一次上面的 ReadIndex 机制来 read，但上面已经说了，这套机制性能也是有保证的。至于为什么我们不在 heartbeat 那边更新 lease，原因就是我们 TiKV 的 Raft 代码想跟 etcd 保持一致，但 etcd 没这个需求，所以我们就做到了外面。

## 小结

在 TiKV 里面，从最开始的 Raft log read，到后面的 Lease Read，我们一步一步的在保证线性一致性的情况下面改进着性能。后面，我们会引入更多的一致性测试 case 来验证整个系统的安全性，当然，也会持续的提升性能。
