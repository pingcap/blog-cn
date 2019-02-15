---
title: TiDB 源码阅读系列文章（十九）tikv-client（下）
author: ['周昱行']
date: 2018-09-26
summary: 本文将继续介绍 tikv-client 里的两个主要的模块——负责处理分布式计算的 copIterator 和执行二阶段提交的 twoPhaseCommitter。
tags: ['源码阅读','TiDB','社区']
---


[上篇文章](http://pingcap.com/blog-cn/tidb-source-code-reading-18/) 中，我们介绍了数据读写过程中 tikv-client 需要解决的几个具体问题，本文将继续介绍 tikv-client 里的两个主要的模块——负责处理分布式计算的 copIterator 和执行二阶段提交的 twoPhaseCommitter。


## copIterator

### copIterator 是什么

在介绍 copIterator 的概念之前，我们需要简单回顾一下前面 [TiDB 源码阅读系列文章（六）](https://pingcap.com/blog-cn/tidb-source-code-reading-6/)中讲过的 distsql 和 coprocessor 的概念以及它们和 SQL 语句的关系。

tikv-server 通过 coprocessor 接口，支持部分 SQL 层的计算能力，大部分只涉及单表数据的常用的算子都可以下推到 tikv-server 上计算，计算下推以后，从存储引擎读取的数据虽然是一样的多，但是通过网络返回的数据会少很多，可以大幅节省序列化和网络传输的开销。

distsql 是位于 SQL 层和 coprocessor 之间的一层抽象，它把下层的 coprocessor 请求封装起来对上层提供一个简单的 `Select` 方法。执行一个单表的计算任务。最上层的 SQL 语句可能会包含 `JOIN`，`SUBQUERY` 等复杂算子，涉及很多的表，而 distsql 只涉及到单个表的数据。一个 distsql 请求会涉及到多个 region，我们要对涉及到的每一个 region 执行一次 coprocessor 请求。

所以它们的关系是这样的，一个 SQL 语句包含多个 distsql 请求，一个 distsql 请求包含多个 coprocessor 请求。

**copIterator 的任务就是实现 distsql 请求，执行所有涉及到的 coprocessor 请求，并依次返回结果。**

### 构造 coprocessor task

一个 distsql 请求需要处理的数据是一个单表上的 index scan 或 table scan，在 Request 包含了转换好的 KeyRange list。接下来，通过 region cache 提供的 [LocateKey](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/region_cache.go#L138) 方法，我们可以找到有哪些 region 包含了一个 key range 范围内的数据。

找到所有 KeyRange 包含的所有的 region 以后，我们需要按照 region 的 range 把 key range list 进行切分，让每个 coprocessor task 里的 key range list 不会超过 region 的范围。

构造出了所有 coprocessor task 之后，下一步就是执行这些 task 了。

### copIterator 的执行模式

为了更容易理解 copIterator 的执行模式，我们先从最简单的实现方式开始， 逐步推导到现在的设计。

copIterator 是 `kv.Response` 接口的实现，需要实现对应 [Next](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L516) 方法，在上层调用 Next  的时候，返回一个 [coprocessor response](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L390:6)，上层通过多次调用 `Next` 方法，获取多个 coprocessor response，直到所有结果获取完。

最简单的实现方式，是在 `Next` 方法里，执行一个 coprocessor task，返回这个 task 的执行结果。

这个执行方式的一个很大的问题，大量时间耗费在等待 coprocessor 请求返回结果，我们需要改进一下。

coprocessor 请求如果是由 `Next` 触发的，每次调用 `Next` 就必须等待一个 `RPC  round trip` 的延迟。我们可以改造成请求在 `Next` 被调用之前触发，这样就能在 Next 被调用的时候，更早拿到结果返回，省掉了阻塞等待的过程。

在 copIterator 创建的时候，我们启动一个后台 worker goroutine 来依次执行所有的 coprocessor task，并把执行结果发送到一个 response channel，这样前台 `Next` 方法只需要从这个 channel 里  receive 一个 coprocessor response 就可以了。如果这个 task 已经执行完成，`Next` 方法可以直接获取到结果，立即返回。

当所有 coprocessor task 被 worker 执行完成的时候，worker 把这个 response channel 关闭，`Next` 方法在 receive channel 的时候发现 channel 已经关闭，就可以返回 `nil response`，表示所有结果都处理完成了。

以上的执行方案还是存在一个问题，就是 coprocessor task 只有一个 worker 在执行，没有并行，性能还是不理想。

为了增大并行度，我们可以构造多个 worker 来执行 task，把所有的 task 发送到一个 task channel，多个 worker 从这一个 channel 读取 task，执行完成后，把结果发到 response channel，通过设置 worker 的数量控制并发度。

这样改造以后，就可以充分的并行执行了，但是这样带来一个新的问题，task 是有序的，但是由于多个 worker 并行执行，返回的 response 顺序是乱序的。对于不要求结果有序的 distsql 请求，这个执行模式是可行的，我们使用这个模式来执行。对于要求结果有序的 distsql 请求，就不能满足要求了，我们需要另一种执行模式。

当 worker 执行完一个 task 之后，当前的做法是把 response 发送到一个全局的 channel 里，如果我们给每一个 task 创建一个 channel，把 response 发送到这个 task 自己的 response channel 里，Next 的时候，就可以按照 task 的顺序获取 response，保证结果的有序。

以上就是 copIterator 最终的执行模式。

### copIterator 实现细节

理解执行模式之后，我们从源码的角度，分析一遍完整的执行流程。

#### 前台执行流程

前台的执行的第一步是 CopClient 的 [Send](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L82) 方法。先根据 distsql 请求里的 `KeyRanges` [构造 coprocessor task](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L238)，用构造好的 task 创建 [copIterator](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L88)，然后调用 copIterator 的 [open](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L438) 方法，启动多个后台 [worker goroutine](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L452)，然后启动一个 [sender](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L454) 用来把 task 丢进 task channel，最后 copIterator 做为 `kv.Reponse` 返回。

前台执行的第二步是多次调用 `kv.Response` 的 `Next` 方法，直到获取所有的 response。

copIterator 在 `Next` 里会根据结果是否有序，选择相应的执行模式，无序的请求会从 [全局 channel 里获取结果](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L526)，有序的请求会在每一个 [task 的 response channel](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L537) 里获取结果。

#### 后台执行流程

[从 task channel 获取到一个 task](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L417) 之后，worker 会执行 [handleTask](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L424) 来发送 RPC 请求，并处理请求的异常，当 region 分裂的时候，我们需要重新构造 [新的 task](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L572)，并重新发送。对于有序的 distsql 请求，分裂后的多个 task 的执行结果需要发送到旧的 task 的 response channel 里，所以一个 task 的 response channel 可能会返回多个 response，发送完成后需要 [关闭 task 的 response channel](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L428)。

## twoPhaseCommitter

### 2PC 简介

2PC 是实现分布式事务的一种方式，保证跨越多个网络节点的事务的原子性，不会出现事务只提交一半的问题。

在 TiDB，使用的 2PC 模型是 Google percolator 模型，简单的理解，percolator 模型和传统的 2PC 的区别主要在于消除了事务管理器的单点，把事务状态信息保存在每个 key 上，大幅提高了分布式事务的线性 scale 能力，虽然仍然存在一个 timestamp oracle 的单点，但是因为逻辑非常简单，而且可以 batch 执行，所以并不会成为系统的瓶颈。

关于 percolator 模型的细节，可以参考这篇文章的介绍 [https://pingcap.com/blog-cn/percolator-and-txn/](https://pingcap.com/blog-cn/percolator-and-txn/)

### 构造 twoPhaseCommitter

当一个事务准备提交的时候，会创建一个 [twoPhaseCommiter](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L62)，用来执行分布式的事务。

构造的时候，需要做以下几件事情

* [从 `memBuffer` 和 `lockedKeys` 里收集所有的 key 和 mutation](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L91)

    `memBuffer` 里的 key 是有序排列的，我们从头遍历 `memBuffer` 可以顺序的收集到事务里需要修改的 key，value 长度为 0 的 entry 表示 `DELETE` 操作，value 长度大于 0 表示 `PUT` 操作，`memBuffer` 里的第一个 key 做为事务的 primary key。`lockKeys` 里保存的是不需要修改，但需要加读锁的 key，也会做为 mutation 的 `LOCK ` 操作，写到 TiKV 上。

* [计算事务的大小是否超过限制](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L132)

    在收集 mutation 的时候，会统计整个事务的大小，如果超过了最大事务限制，会返回报错。

    太大的事务可能会让 TiKV 集群压力过大，执行失败并导致集群不可用，所以要对事务的大小做出硬性的限制。

* [计算事务的 TTL 时间](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L164)

    如果一个事务的 key 通过 `prewrite` 加锁后，事务没有执行完，tidb-server 就挂掉了，这时候集群内其他 tidb-server 是无法读取这个 key 的，如果没有 TTL，就会死锁。设置了 TTL 之后，读请求就可以在 TTL 超时之后执行清锁，然后读取到数据。

    我们计算一个事务的超时时间需要考虑正常执行一个事务需要花费的时间，如果太短会出现大的事务无法正常执行完的问题，如果太长，会有异常退出导致某个 key 长时间无法访问的问题。所以使用了这样一个算法，TTL 和事务的大小的平方根成正比，并控制在一个最小值和一个最大值之间。

### execute

在 twoPhaseCommiter 创建好以后，下一步就是执行 [execute](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L562) 函数。

在 `execute` 函数里，需要在 `defer` 函数里执行 [cleanupKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L572)，在事务没有成功执行的时候，清理掉多余的锁，如果不做这一步操作，残留的锁会让读请求阻塞，直到 TTL 过期才会被清理。第一步会执行 [prewriteKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L585)，如果成功，会从 PD 获取一个 `commitTS` 用来执行 `commit` 操作。取到了 `commitTS` 之后，还需要做以下验证:

* `commitTS` 比 `startTS` 大

* schema 没有过期

* 事务的执行时间没有过长

* 如果没有通过检查，事务会失败报错。

通过检查之后，执行最后一步 [commitKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L620)，如果没有错误，事务就提交完成了。

当 `commitKeys` 请求遇到了网络超时，那么这个事务是否已经提交是不确定的，这时候不能执行 `cleanupKeys` 操作，否则就破坏了事务的一致性。我们对这种情况返回一个特殊的 [undetermined error](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L625)，让上层来处理。上层会在遇到这种 error 的时候，把连接断开，而不是返回给用户一个执行失败的错误。

[prewriteKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L533),  [commitKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L537) 和 [cleanupKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L541) 有很多相同的逻辑，需要把 keys 根据 region 分成 batch，然后对每个 batch 执行一次 RPC。

当 RPC 返回 region 过期的错误时，我们需要把这个 region 上的 keys 重新分成 batch，发送 RPC 请求。

这部分逻辑我们把它抽出来，放在 [doActionOnKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L191) 和 [doActionOnBatches](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L239) 里，并实现 [prewriteSinlgeBatch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L319)，[commitSingleBatch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L421)，[cleanupSingleBatch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L497) 函数，用来执行单个 batch 的 RPC 请求。

虽然大部分逻辑是相同的，但是不同的请求在执行顺序上有一些不同，在 `doActionOnKeys` 里需要特殊的判断和处理。

* `prewrite` 分成的多个 batch 需要同步并行的执行。

* `commit` 分成的多个 batch 需要先执行第一个 batch，成功后再异步并行执行其他的 batch。

* `cleanup` 分成的多个 batch 需要异步并行执行。

[doActionOnBatches](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L239:29) 会开启多个 goroutines 并行的执行多个 batch，如果遇到了 error，会把其他正在执行的 `context cancel` 掉，然后返回第一个遇到的 error。

执行 `prewriteSingleBatch` 的时候，有可能会遇到 region 分裂错误，这时候 batch 里的 key 就不再是一个 region 上的 key 了，我们会在这里递归的调用 [prewriteKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L352)，重新走一遍拆分 batch 然后执行 `doActionOnBatch` 和 `prewriteSingleBatch` 的流程。这部分逻辑在 `commitSingleBatch` 和 `cleanupSingleBatch` 里也都有。

twoPhaseCommitter 包含的逻辑只是事务模型的一小部分，主要的逻辑在 tikv-server 端，超出了这篇文章的范围，就不在这里详细讨论了。
