---
title: TiKV 功能介绍 - Placement Driver
author: ['唐刘']
date: 2017-01-08
summary: Placement Driver (后续以 PD 简称) 是 TiDB 里面全局中心总控节点，它负责整个集群的调度，负责全局 ID 的生成，以及全局时间戳 TSO 的生成等。PD 还保存着整个集群 TiKV 的元信息，负责给 client 提供路由功能。
tags: ['TiKV', 'PD', '集群调度']
---


## 介绍

Placement Driver (后续以 PD 简称) 是 TiDB 里面全局中心总控节点，它负责整个集群的调度，负责全局 ID 的生成，以及全局时间戳 TSO 的生成等。PD 还保存着整个集群 TiKV 的元信息，负责给 client 提供路由功能。

作为中心总控节点，PD 通过集成 [etcd](https://github.com/coreos/etcd) ，自动的支持 auto failover，无需担心单点故障问题。同时，PD 也通过 etcd 的 raft，保证了数据的强一致性，不用担心数据丢失的问题。

在架构上面，PD 所有的数据都是通过 TiKV 主动上报获知的。同时，PD 对整个 TiKV 集群的调度等操作，也只会在 TiKV 发送 heartbeat 命令的结果里面返回相关的命令，让 TiKV 自行去处理，而不是主动去给 TiKV 发命令。这样设计上面就非常简单，我们完全可以认为 PD 是一个无状态的服务（当然，PD 仍然会将一些信息持久化到 etcd），所有的操作都是被动触发，即使 PD 挂掉，新选出的 PD leader 也能立刻对外服务，无需考虑任何之前的中间状态。

## 初始化

PD 集成了 etcd，所以通常，我们需要启动至少三个副本，才能保证数据的安全。现阶段 PD 有集群启动方式，`initial-cluster` 的静态方式以及 `join` 的动态方式。

在继续之前，我们需要了解下 etcd 的端口，在 etcd 里面，默认要监听 2379 和 2380 两个端口。2379 主要是 etcd 用来处理外部请求用的，而 2380 则是 etcd peer 之间相互通信用的。

假设现在我们有三个 pd，分别为 pd1，pd2，pd3，分别在 host1，host2，host3 上面。

对于静态初始化，我们直接在三个 PD 启动的时候，给 `initial-cluster` 设置 `pd1=http://host1:2380,pd2=http://host2:2380,pd3=http://host3:2380`。

对于动态初始化，我们先启动 pd1，然后启动 pd2，加入到 pd1 的集群里面，`join` 设置为 `pd1=http://host1:2379`。然后启动 pd3，加入到 pd1，pd2 形成的集群里面， `join` 设置为 `pd1=http://host1:2379`。

可以看到，静态初始化和动态初始化完全走的是两个端口，而且这两个是互斥的，也就是我们只能使用一种方式来初始化集群。etcd 本身只支持 `initial-cluster` 的方式，但为了方便，PD 同时也提供了 `join` 的方式。

`join` 主要是用了 etcd 自身提供的 member 相关 API，包括 add member，list member 等，所以我们使用 2379 端口，因为需要将命令发到 etcd 去执行。而 `initial-cluster` 则是 etcd 自身的初始化方式，所以使用的 2380 端口。

相比于 `initial-cluster`，`join` 需要考虑非常多的 case（在 `server/join.go` `prepareJoinCluster` 函数里面有详细的解释），但 `join` 的使用非常自然，后续我们会考虑去掉 `initial-cluster` 的初始化方案。

## 选举

当 PD 启动之后，我们就需要选出一个 leader 对外提供服务。虽然 etcd 自身也有 raft leader，但我们还是觉得使用自己的 leader，也就是 PD 的 leader 跟 etcd 自己的 leader 是不一样的。

当 PD 启动之后，Leader 的选举如下：

1. 检查当前集群是不是有 leader，如果有 leader，就 watch 这个 leader，只要发现 leader 掉了，就重新开始 1。
2. 如果没有 leader，开始 campaign，创建一个 Lessor，并且通过 etcd 的事务机制写入相关信息，如下：

    ```go
    // Create a lessor.
    ctx, cancel := context.WithTimeout(s.client.Ctx(), requestTimeout)
	leaseResp, err := lessor.Grant(ctx, s.cfg.LeaderLease)
	cancel()

    // The leader key must not exist, so the CreateRevision is 0.
	resp, err := s.txn().
		If(clientv3.Compare(clientv3.CreateRevision(leaderKey), "=", 0)).
		Then(clientv3.OpPut(leaderKey, s.leaderValue, clientv3.WithLease(clientv3.LeaseID(leaseResp.ID)))).
		Commit()
    ```

    如果 leader key 的 CreateRevision 为 0，表明其他 PD 还没有写入，那么我就可以将我自己的 leader 相关信息写入，同时会带上一个 Lease。如果事务执行失败，表明其他的 PD 已经成为了 leader，那么就重新回到 1。

3. 成为 leader 之后，我们对定期进行保活处理:

    ```go
    // Make the leader keepalived.
	ch, err := lessor.KeepAlive(s.client.Ctx(), clientv3.LeaseID(leaseResp.ID))
	if err != nil {
		return errors.Trace(err)
	}
    ```

    当 PD 崩溃，原先写入的 leader key 会因为 lease 到期而自动删除，这样其他的 PD 就能 watch 到，重新开始选举。

4. 初始化 raft cluster，主要是从 etcd 里面重新载入集群的元信息。拿到最新的 TSO 信息：

    ```go
    // Try to create raft cluster.
	err = s.createRaftCluster()
	if err != nil {
		return errors.Trace(err)
	}

	log.Debug("sync timestamp for tso")
	if err = s.syncTimestamp(); err != nil {
		return errors.Trace(err)
	}
    ```

5. 所有做完之后，开始定期更新 TSO，监听 lessor 是否过期，以及外面是否主动退出：

    ```go
    for {
		select {
		case _, ok := <-ch:
			if !ok {
				log.Info("keep alive channel is closed")
				return nil
			}
		case <-tsTicker.C:
			if err = s.updateTimestamp(); err != nil {
				return errors.Trace(err)
			}
		case <-s.client.Ctx().Done():
			return errors.New("server closed")
		}
	}
    ```

### TSO

前面我们说到了 TSO，TSO 是一个全局的时间戳，它是 TiDB 实现分布式事务的基石。所以对于 PD 来说，我们首先要保证它能快速大量的为事务分配 TSO，同时也需要保证分配的 TSO 一定是单调递增的，不可能出现回退的情况。

TSO 是一个 int64 的整形，它由 physical time + logical time 两个部分组成。Physical time 是当前 unix time 的毫秒时间，而 logical time 则是一个最大 `1 << 18` 的计数器。也就是说 1ms，PD 最多可以分配 262144 个 TSO，这个能满足绝大多数情况了。

对于 TSO 的保存于分配，PD 会做如下处理：

1. 当 PD 成为 leader 之后，会从 etcd 上面获取上一次保存的时间，如果发现本地的时间比这个小，则会继续等待直到当前的时间大于这个值：

    ```go
    last, err := s.loadTimestamp()
    if err != nil {
    	return errors.Trace(err)
    }

    var now time.Time

    for {
    	now = time.Now()
    	if wait := last.Sub(now) + updateTimestampGuard; wait > 0 {
    		log.Warnf("wait %v to guarantee valid generated timestamp", wait)
    		time.Sleep(wait)
    		continue
    	}
    	break
    }
    ```

2. 当 PD 能分配 TSO 之后，首先会向 etcd 申请一个最大的时间，譬如，假设当前时间是 t1，每次最多能申请 3s 的时间窗口，PD 会向 etcd 保存 t1 + 3s 的时间值，然后 PD 就能在内存里面直接使用这一段时间窗口.当当前的时间 t2 大于 t1 + 3s 之后，PD 就会在向 etcd 继续更新为 t2 + 3s：

    ```go
    if now.Sub(s.lastSavedTime) >= 0 {
    	last := s.lastSavedTime
    	save := now.Add(s.cfg.TsoSaveInterval.Duration)
    	if err := s.saveTimestamp(save); err != nil {
    		return errors.Trace(err)
    	}
    }
    ```

    这么处理的好处在于，即使 PD 当掉，新启动的 PD 也会从上一次保存的最大的时间之后开始分配 TSO，也就是 1 处理的情况。

3. 因为 PD 在内存里面保存了一个可分配的时间窗口，所以外面请求 TSO 的时候，PD 能直接在内存里面计算 TSO 并返回。

    ```go
    resp := pdpb.Timestamp{}
    for i := 0; i < maxRetryCount; i++ {
    	current, ok := s.ts.Load().(*atomicObject)
    	if !ok {
    		log.Errorf("we haven't synced timestamp ok, wait and retry, retry count %d", i)
    		time.Sleep(200 * time.Millisecond)
    		continue
    	}

    	resp.Physical = current.physical.UnixNano() / int64(time.Millisecond)
    	resp.Logical = atomic.AddInt64(&current.logical, int64(count))
    	if resp.Logical >= maxLogical {

    		time.Sleep(updateTimestampStep)
    		continue
    	}
    	return resp, nil
    }
    ```

    因为是在内存里面计算的，所以性能很高，我们自己内部测试每秒能分配百万级别的 TSO。

4. 如果 client 每次事务都向 PD 来请求一次 TSO，每次 RPC 的开销也是非常大的，所以 client 会批量的向 PD 获取 TSO。client 会首先收集一批事务的 TSO 请求，譬如 n 个，然后直接向 PD 发送命令，参数就是 n，PD 收到命令之后，会生成 n 个 TSO 返回给客户端。

## 心跳

在最开始我们说过，PD 所有关于集群的数据都是由 TiKV 主动心跳上报的，PD 对 TiKV 的调度也是在心跳的时候完成的。通常 PD 会处理两种心跳，一个是 TiKV 自身 store 的心跳，而另一个则是 store 里面 region 的 leader peer 上报的心跳。

对于 store 的心跳，PD 在 `handleStoreHeartbeat` 函数里面处理，主要就是将心跳里面当前的 store 的一些状态缓存到 cache 里面。store 的状态包括该 store 有多少个 region，有多少个 region 的 leader peer 在该 store 上面等，这些信息都会用于后续的调度。

对于 region 的心跳，PD 在 `handleRegionHeartbeat` 里面处理。这里需要注意，只有 leader peer 才会去上报所属 region 的信息，follower peer 是不会上报的。收到 region 的心跳之后，首先 PD 也会将其放入 cache 里面，如果 PD 发现 region 的 epoch 有变化，就会将这个 region 的信息也保存到 etcd 里面。然后，PD 会对这个 region 进行具体的调度，譬如发现 peer 数目不够，添加新的 peer，或者有一个 peer 已经坏了，删除这个 peer 等，详细的调度实现，我们会在后续讨论。

这里再说一下 region 的 epoch，在 region 的 epoch 里面，有 `conf_ver` 和 `version`，分别表示这个 region 不同的版本状态。如果一个 region 发生了 membership changes，也就是新增或者删除了 peer，`conf_ver` 会加 1，如果 region 发生了 `split` 或者 `merge`，则 `version` 加 1。

无论是 PD 还是在 TiKV，我们都是通过 epoch 来判断 region 是否发生了变化，从而拒绝掉一些危险的操作。譬如 region 已经发生了分裂，`version` 变成了 2，那么如果这时候有一个写请求带上的 `version` 是 1， 我们就会认为这个请求是 stale，会直接拒绝掉。因为 `version` 变化表明 region 的范围已经发生了变化，很有可能这个 stale 的请求需要操作的 key 是在之前的 region range 里面而没在新的 range 里面。


## Split / Merge

前面我们说了，PD 会在 region 的 heartbeat 里面对 region 进行调度，然后直接在 heartbeat 的返回值里面带上相关的调度信息，让 TiKV 自己去处理，TiKV 处理完成之后，通过下一个 heartbeat 重新上报，PD 就能知道是否调度成功了。

对于 membership changes，比较容易，因为我们有最大副本数的配置，假设三个，那么当 region 的心跳上来，发现只有两个 peer，那么就 add peer，如果有四个 peer，就 remove peer。而对于 region 的 split / merge，则情况稍微要复杂一点，但也比较简单。注意，现阶段，我们只支持 split，merge 处于开发阶段，没对外发布，所以这里仅仅以 split 举例：

1. 在 TiKV 里面，leader peer 会定期检查 region 所占用的空间是否超过某一个阀值，假设我们设置 region 的 size 为 64MB，如果一个 region 超过了 96MB， 就需要分裂。
2. Leader peer 会首先向 PD 发送一个请求分裂的命令，PD 在 `handleAskSplit` 里面处理，因为我们是一个 region 分裂成两个，对于这两个新分裂的 region，一个会继承之前 region 的所有的元信息，而另一个相关的信息，譬如 region ID，新的 peer ID，则需要 PD 生成，并将其返回给 leader。
3. Leader peer 写入一个 split raft log，在 apply 的时候执行，这样 region 就分裂成了两个。
4. 分裂成功之后，TiKV 告诉 PD，PD 就在 `handleReportSplit` 里面处理，更新 cache 相关的信息，并持久化到 etcd。

## 路由

因为 PD 保存了所有 TiKV 的集群信息，自然对 client 提供了路由的功能。假设 client 要对 `key` 写入一个值。

1. client 先从 PD 获取 `key` 属于哪一个 region，PD 将这个 region 相关的元信息返回。
2. client 自己 cache，这样就不需要每次都从 PD 获取。然后直接给 region 的 leader peer 发送命令。
3. 有可能 region 的 leader 已经漂移到其他 peer，TiKV 会返回 `NotLeader` 错误，并带上新的 leader 的地址，client 在 cache 里面更新，并重新向新的 leader 发送请求。
4. 也有可能 region 的 version 已经变化，譬如 split 了，这时候，`key` 可能已经落入了新的 region 上面，client 会收到 `StaleCommand` 的错误，于是重新从 PD 获取，进入状态 1。

## 小结

PD 作为 TiDB 集群的中心调度模块，在设计上面，我们尽量保证无状态，方便扩展。本篇文章主要介绍了 PD 是如何跟 TiKV，TiDB 协作交互的。后面，我们会详细地介绍核心调度功能，也就是 PD 是如何控制整个集群的。
