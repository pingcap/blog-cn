---
title: TiKV 源码阅读三部曲（二）读流程
author: ['谭新宇']
date: 2022-10-26
summary: 本文是 TiKV 源码阅读三部曲第二篇，主要介绍 TiKV 中一条读请求的全链路流程。
tags: ["TiKV 源码阅读"]
---

[TiKV](https://github.com/tikv/tikv) 是一个支持事务的分布式 Key-Value 数据库，目前已经是 [CNCF 基金会](https://www.cncf.io/projects/) 的顶级项目。

作为一个新同学，需要一定的前期准备才能够有能力参与 TiKV 社区的代码开发，包括但不限于学习 Rust 语言，理解 TiKV 的原理和在前两者的基础上了解熟悉 TiKV 的源码。

[TiKV 官方源码解析文档](/blog/?tag=TiKV%20源码解析) 详细地介绍了 TiKV 3.x 版本重要模块的设计要点，主要流程和相应代码片段，是学习 TiKV 源码必读的学习资料。当前 TiKV 已经迭代到了 6.x 版本，不仅引入了很多新的功能和优化，而且对源码也进行了多次重构，因而一些官方源码解析文档中的代码片段已经不复存在，这使得读者在阅读源码解析文档时无法对照最新源码加深理解；此外尽管 TiKV 官方源码解析文档系统地介绍了若干重要模块的工作，但并没有将读写流程全链路串起来去介绍经过的模块和对应的代码片段，实际上尽快地熟悉读写流程全链路会更利于新同学从全局角度理解代码。

基于以上存在的问题，笔者将基于 6.1 版本的源码撰写三篇博客，分别介绍以下三个方面：

- **[TiKV 源码阅读三部曲（一）重要模块](/blog/tikv-source-code-reading-module)**：TiKV 的基本概念，TiKV 读写路径上的三个重要模块（KVService，Storage，RaftStore）和断点调试 TiKV 学习源码的方案
- **TiKV 源码阅读三部曲（二）读流程**：TiKV 中一条读请求的全链路流程
- **TiKV 源码阅读三部曲（三）写流程**：TiKV 中一条写请求的全链路流程

希望此三篇博客能够帮助对 TiKV 开发感兴趣的新同学尽快了解 TiKV 的 codebase。

本文为第二篇博客，将主要介绍 TiKV 中一条读请求的全链路流程。

## 读流程
[TiKV 源码解析系列文章（十九）read index 和 local read 情景分析](/blog/tikv-source-code-reading-19) 介绍了 TiKV 3.x 版本的 ReadIndex/LeaseRead 实现方案。

本小节将在 TiKV 6.1 版本的源码基础上，以一条读请求为例，介绍当前版本读请求的全链路执行流程。

前文已经提到，可以从 [kvproto](https://github.com/pingcap/kvproto/blob/master/proto/tikvpb.proto#L20) 对应的 `service Tikv` 中了解当前 TiKV 支持的 RPC 接口。

经过简单整理，常用的读接口如下：

```rust
// Key/value store API for TiKV.
service Tikv { 

    rpc KvGet(kvrpcpb.GetRequest) returns (kvrpcpb.GetResponse) {}
    rpc KvScan(kvrpcpb.ScanRequest) returns (kvrpcpb.ScanResponse) {}
    rpc KvBatchGet(kvrpcpb.BatchGetRequest) returns (kvrpcpb.BatchGetResponse) {}

    rpc RawGet(kvrpcpb.RawGetRequest) returns (kvrpcpb.RawGetResponse) {}
    rpc RawBatchGet(kvrpcpb.RawBatchGetRequest) returns (kvrpcpb.RawBatchGetResponse) {}
    rpc RawScan(kvrpcpb.RawScanRequest) returns (kvrpcpb.RawScanResponse) {}
    rpc RawBatchScan(kvrpcpb.RawBatchScanRequest) returns (kvrpcpb.RawBatchScanResponse) {}

    ...
}
```

以下将以最常用的 KvGet 接口为例介绍读流程，其他的读接口所经过的模块大致相似，之后也可以用断点调试的方案去自行阅读。

### KVService

在 KVService 中， handle_request 宏将业务逻辑封装到了 future_get 函数中。在 future_get 函数中，主要使用了 `storage.get(req.take_context(), Key::from_raw(req.get_key()), req.get_version().into())` 函数将请求路由到 Storage 模块去执行。

为了可观测性，当前 TiKV 在读写关键路径上加了很多全局和 request 级别的 metric，这一定程度上影响了刚开始阅读代码的体验。其实刚开始熟悉代码时只需要关注核心逻辑即可，metric 相关的代码可以先不用细究。

```rust
impl<T: RaftStoreRouter<E::Local> + 'static, E: Engine, L: LockManager, F: KvFormat> Tikv
    for Service<T, E, L, F>
{
    handle_request!(kv_get, future_get, GetRequest, GetResponse, has_time_detail);
}

fn future_get<E: Engine, L: LockManager, F: KvFormat>(
    storage: &Storage<E, L, F>,
    mut req: GetRequest,
) -> impl Future<Output = ServerResult<GetResponse>> {

    ...

    let v = storage.get(
        req.take_context(),
        Key::from_raw(req.get_key()),
        req.get_version().into(),
    );

    async move {
        let v = v.await;
        
        ...
        
        Ok(resp)
    }
}
```

### Storage

在 Storage 模块的 get 函数中，所有的 task 都会被 spawn 到 readPool 中执行，具体执行的任务主要包含以下两个工作：

- 使用 `Self::with_tls_engine(|engine| Self::snapshot(engine, snap_ctx)).await?` 获取 snapshot
- 使用 `snap_store.get(&key, &mut statistics)` 基于获取到的 snapshot 获取符合对应事务语义的数据

第二个工作比较简单，本小节不再赘述，以下主要介绍第一个工作的具体代码流程。

```rust
/// Get value of the given key from a snapshot.
///
/// Only writes that are committed before `start_ts` are visible.
pub fn get(
    &self,
    mut ctx: Context,
    key: Key,
    start_ts: TimeStamp,
) -> impl Future<Output = Result<(Option<Value>, KvGetStatistics)>> {

    ...

    let res = self.read_pool.spawn_handle(
        async move {

            ...

            let snap_ctx = prepare_snap_ctx(
                &ctx,
                iter::once(&key),
                start_ts,
                &bypass_locks,
                &concurrency_manager,
                CMD,
            )?;
            let snapshot =
                Self::with_tls_engine(|engine| Self::snapshot(engine, snap_ctx)).await?;

            {
                let begin_instant = Instant::now();
                let stage_snap_recv_ts = begin_instant;
                let buckets = snapshot.ext().get_buckets();
                let mut statistics = Statistics::default();
                let result = Self::with_perf_context(CMD, || {
                    let _guard = sample.observe_cpu();
                    let snap_store = SnapshotStore::new(
                        snapshot,
                        start_ts,
                        ctx.get_isolation_level(),
                        !ctx.get_not_fill_cache(),
                        bypass_locks,
                        access_locks,
                        false,
                    );
                    snap_store
                    .get(&key, &mut statistics)
                    // map storage::txn::Error -> storage::Error
                    .map_err(Error::from)
                    .map(|r| {
                        KV_COMMAND_KEYREAD_HISTOGRAM_STATIC.get(CMD).observe(1_f64);
                        r
                    })
                });
                
                ...
        
                Ok((
                    result?,
                    KvGetStatistics {
                        stats: statistics,
                        latency_stats,
                    },
                ))
            }
        }
        .in_resource_metering_tag(resource_tag),
        priority,
        thread_rng().next_u64(),
    );
    async move {
        res.map_err(|_| Error::from(ErrorInner::SchedTooBusy))
            .await?
    }
}
```

对于 `Self::snapshot(engine, snap_ctx)` 函数，其会经由 `storage::snapshot -> kv::snapshot -> raftkv::async_snapshot -> raftkv::exec_snapshot` 的调用链来到 `ServerRaftStoreRouter::read` 函数中。

```rust
/// Get a snapshot of `engine`.
fn snapshot(
    engine: &E,
    ctx: SnapContext<'_>,
) -> impl std::future::Future<Output = Result<E::Snap>> {
    kv::snapshot(engine, ctx)
        .map_err(txn::Error::from)
        .map_err(Error::from)
}

/// Get a snapshot of `engine`.
pub fn snapshot<E: Engine>(
    engine: &E,
    ctx: SnapContext<'_>,
) -> impl std::future::Future<Output = Result<E::Snap>> {
    let begin = Instant::now();
    let (callback, future) =
        tikv_util::future::paired_must_called_future_callback(drop_snapshot_callback::<E>);
    let val = engine.async_snapshot(ctx, callback);
    // make engine not cross yield point
    async move {
        val?; // propagate error
        let result = future
            .map_err(|cancel| Error::from(ErrorInner::Other(box_err!(cancel))))
            .await?;
        with_tls_tracker(|tracker| {
            tracker.metrics.get_snapshot_nanos += begin.elapsed().as_nanos() as u64;
        });
        fail_point!("after-snapshot");
        result
    }
}

fn async_snapshot(&self, mut ctx: SnapContext<'_>, cb: Callback<Self::Snap>) -> kv::Result<()> {
    
    ...

    self.exec_snapshot(
        ctx,
        req,
        Box::new(move |res| match res {
            ...
        }),
    )
    .map_err(|e| {
        let status_kind = get_status_kind_from_error(&e);
        ASYNC_REQUESTS_COUNTER_VEC.snapshot.get(status_kind).inc();
        e.into()
    })
}

fn exec_snapshot(
    &self,
    ctx: SnapContext<'_>,
    req: Request,
    cb: Callback<CmdRes<E::Snapshot>>,
) -> Result<()> {

    ...
    
    let mut cmd = RaftCmdRequest::default();
    cmd.set_header(header);
    cmd.set_requests(vec![req].into());
    self.router
        .read(
            ctx.read_id,
            cmd,
            StoreCallback::read(Box::new(move |resp| {
                cb(on_read_result(resp).map_err(Error::into));
            })),
        )
        .map_err(From::from)
}

impl<EK: KvEngine, ER: RaftEngine> LocalReadRouter<EK> for ServerRaftStoreRouter<EK, ER> {
    fn read(
        &self,
        read_id: Option<ThreadReadId>,
        req: RaftCmdRequest,
        cb: Callback<EK::Snapshot>,
    ) -> RaftStoreResult<()> {
        let mut local_reader = self.local_reader.borrow_mut();
        local_reader.read(read_id, req, cb);
        Ok(())
    }
}
```

在 `ServerRaftStoreRouter::read` 函数中，其会调用 `local_reader` 的 `read` 函数，并进而路由到 `LocalReader::propose_raft_command` 函数。在该函数中，会使用 `LocalReader::pre_propose_raft_command` 函数来判断是否能够 ReadLocal，如果可以则直接获取本地引擎的 snapshot 并执行 callback 返回即可，否则便调用 `redirect` 函数连带 callback 路由到 RaftBatchSystem 的对应 normal 状态机中去执行 ReadIndex 读，之后本线程不再处理该任务。

```rust
#[inline]
pub fn read(
    &mut self,
    read_id: Option<ThreadReadId>,
    req: RaftCmdRequest,
    cb: Callback<E::Snapshot>,
) {
    self.propose_raft_command(read_id, req, cb);
    maybe_tls_local_read_metrics_flush();
}

pub fn propose_raft_command(
    &mut self,
    mut read_id: Option<ThreadReadId>,
    req: RaftCmdRequest,
    cb: Callback<E::Snapshot>,
) {
    match self.pre_propose_raft_command(&req) {
        Ok(Some((mut delegate, policy))) => {
            let delegate_ext: LocalReadContext<'_, E>;
            let mut response = match policy {
                // Leader can read local if and only if it is in lease.
                RequestPolicy::ReadLocal => {
                 
                    ...

                    let region = Arc::clone(&delegate.region);
                    let response =
                        delegate.execute(&req, &region, None, read_id, Some(delegate_ext));
                    // Try renew lease in advance
                    delegate.maybe_renew_lease_advance(&self.router, snapshot_ts);
                    response
                }
                // Replica can serve stale read if and only if its `safe_ts` >= `read_ts`
                RequestPolicy::StaleRead => {
               
                    ...

                    let region = Arc::clone(&delegate.region);
                    // Getting the snapshot
                    let response =
                        delegate.execute(&req, &region, None, read_id, Some(delegate_ext));

                    ...
                    
                }
                _ => unreachable!(),
            };
           
            ...
            
            cb.invoke_read(response);
        }
        // Forward to raftstore.
        Ok(None) => self.redirect(RaftCommand::new(req, cb)),
        Err(e) => {
            let mut response = cmd_resp::new_error(e);
            if let Some(delegate) = self.delegates.get(&req.get_header().get_region_id()) {
                cmd_resp::bind_term(&mut response, delegate.term);
            }
            cb.invoke_read(ReadResponse {
                response,
                snapshot: None,
                txn_extra_op: TxnExtraOp::Noop,
            });
        }
    }
}
```

需要注意的是，在此处能否 ReadLocal 的判断是可以并行的，也就是乐观情况下并行的读请求可以并行获取底层引擎的 snapshot，不需要经过 RaftBatchSystem 。

那么到底什么时候可以直接读取 snapshot 而不需要经过 RaftStore 走一轮 ReadIndex 来处理呢？原理就是 Lease 机制，可以先简单阅读一下 [TiKV Lease Read 的功能介绍](/blog/lease-read)。

接着让我们回到 `LocalReader::pre_propose_raft_command` 函数，其会进行一系列的检查（此处已略去），如果皆通过则会进一步调用 `inspector.inspect(req)` 函数，在其内部，其会进行一系列的判断并返回是否可以 ReadLocal。

- `req.get_header().get_read_quorum()`：如果该请求明确要求需要用 read index 方式处理，所以返回 ReadIndex。
- `self.has_applied_to_current_term()`：如果该 leader 尚未 apply 到它自己的 term，则使用 ReadIndex 处理，这是 Raft 有关线性一致性读的一个 corner case。
- `self.inspect_lease()`：如果该 leader 的 lease 已经过期或者不确定，说明可能出现了一些问题，比如网络不稳定，心跳没成功等，此时使用 ReadIndex 处理，否则便可以使用 ReadLocal 处理。

```rust
pub fn pre_propose_raft_command(
    &mut self,
    req: &RaftCmdRequest,
) -> Result<Option<(D, RequestPolicy)>> {
    
    ...

    match inspector.inspect(req) {
        Ok(RequestPolicy::ReadLocal) => Ok(Some((delegate, RequestPolicy::ReadLocal))),
        Ok(RequestPolicy::StaleRead) => Ok(Some((delegate, RequestPolicy::StaleRead))),
        // It can not handle other policies.
        Ok(_) => Ok(None),
        Err(e) => Err(e),
    }
}

fn inspect(&mut self, req: &RaftCmdRequest) -> Result<RequestPolicy> {

    ...

    fail_point!("perform_read_index", |_| Ok(RequestPolicy::ReadIndex));

    let flags = WriteBatchFlags::from_bits_check(req.get_header().get_flags());
    if flags.contains(WriteBatchFlags::STALE_READ) {
        return Ok(RequestPolicy::StaleRead);
    }

    if req.get_header().get_read_quorum() {
        return Ok(RequestPolicy::ReadIndex);
    }

    // If applied index's term is differ from current raft's term, leader transfer
    // must happened, if read locally, we may read old value.
    if !self.has_applied_to_current_term() {
        return Ok(RequestPolicy::ReadIndex);
    }

    // Local read should be performed, if and only if leader is in lease.
    // None for now.
    match self.inspect_lease() {
        LeaseState::Valid => Ok(RequestPolicy::ReadLocal),
        LeaseState::Expired | LeaseState::Suspect => {
            // Perform a consistent read to Raft quorum and try to renew the leader lease.
            Ok(RequestPolicy::ReadIndex)
        }
    }
}
```

乐观情况下的 ReadLocal 流程我们已经了解，接下来让我们看看 ReadIndex 在 RaftStore 中的执行路径。

### RaftStore

前文已经介绍过 RaftBatchSystem 的大体框架，我们已知会有多个 PollHandler 线程调用 poll 函数进入长期循环来事件驱动并动态均衡地管理所有 normal 状态机。

当 ReadIndex 请求被路由到 RaftBatchSystem 中的对应 normal 状态机后，某个 PollHandler 会在接下来的一次 loop 中处理该状态机的消息。

直接定位到 `RaftPoller` 的 `handle_normal` 函数。可以看到，其会首先尝试获取 `messages_per_tick` 次路由到该状态机的消息，接着调用 `PeerFsmDelegate::handle_msgs` 函数进行处理，

这里只列出了我们需要关注的几种消息类型：

- RaftMessage: 其他 Peer 发送过来 Raft 消息，包括心跳、日志、投票消息等。
- RaftCommand: 上层提出的 proposal，其中包含了需要通过 Raft 同步的操作，以及操作成功之后需要调用的 callback 函数。ReadIndex 请求便是一种特殊的 proposal。
- ApplyRes: ApplyFsm 在将日志应用到状态机之后发送给 PeerFsm 的消息，用于在进行操作之后更新某些内存状态。

```rust
impl<EK: KvEngine, ER: RaftEngine, T: Transport> PollHandler<PeerFsm<EK, ER>, StoreFsm<EK>>
    for RaftPoller<EK, ER, T>
{
    fn handle_normal(
        &mut self,
        peer: &mut impl DerefMut<Target = PeerFsm<EK, ER>>,
    ) -> HandleResult {
        let mut handle_result = HandleResult::KeepProcessing;

        ...

        while self.peer_msg_buf.len() < self.messages_per_tick {
            match peer.receiver.try_recv() {
                // TODO: we may need a way to optimize the message copy.
                Ok(msg) => {
                    ...
                    self.peer_msg_buf.push(msg);
                }
                Err(TryRecvError::Empty) => {
                    handle_result = HandleResult::stop_at(0, false);
                    break;
                }
                Err(TryRecvError::Disconnected) => {
                    peer.stop();
                    handle_result = HandleResult::stop_at(0, false);
                    break;
                }
            }
        }

        let mut delegate = PeerFsmDelegate::new(peer, &mut self.poll_ctx);
        delegate.handle_msgs(&mut self.peer_msg_buf);
        // No readiness is generated and using sync write, skipping calling ready and
        // release early.
        if !delegate.collect_ready() && self.poll_ctx.sync_write_worker.is_some() {
            if let HandleResult::StopAt { skip_end, .. } = &mut handle_result {
                *skip_end = true;
            }
        }

        handle_result
    }
}

impl<'a, EK, ER, T: Transport> PeerFsmDelegate<'a, EK, ER, T>
where
    EK: KvEngine,
    ER: RaftEngine,
{
    pub fn handle_msgs(&mut self, msgs: &mut Vec<PeerMsg<EK>>) {
        for m in msgs.drain(..) {
            match m {
                PeerMsg::RaftMessage(msg) => {
                    if let Err(e) = self.on_raft_message(msg) {
                        error!(%e;
                            "handle raft message err";
                            "region_id" => self.fsm.region_id(),
                            "peer_id" => self.fsm.peer_id(),
                        );
                    }
                }
                PeerMsg::RaftCommand(cmd) => {
                        ...
                        self.propose_raft_command(
                            cmd.request,
                            cmd.callback,
                            cmd.extra_opts.disk_full_opt,
                        );
                    }
                }
                PeerMsg::ApplyRes { res } => {
                    self.on_apply_res(res);
                }
                ...
            }
        }
}
```

对于 ReadIndex 请求，其会进入 `PeerMsg::RaftCommand(cmd)` 分支，进而以 `PeerFsmDelegate::propose_raft_command -> PeerFsmDelegate::propose_raft_command_internal` 的调用链走到 `store::propose` 函数中，在该函数中，会再进行一次 `self.inspect()`，如果此时 Leader 的 lease 已经稳定，则会调用 `read_local` 函数直接获取引擎的 snapshot 并执行 callback 返回，否则调用 `read_index` 函数执行 ReadIndex 流程。

在 read_index 函数中，ReadIndex 请求连带 callback 会被构建成一个 ReadIndexRequest 被 push 到 pending_reads 即一个 ReadIndexQueue 中，之后当前线程即可结束本轮流程，之后的事件会进而触发该 ReadIndexRequest 的执行。

```rust
pub fn propose<T: Transport>(
    &mut self,
    ctx: &mut PollContext<EK, ER, T>,
    mut cb: Callback<EK::Snapshot>,
    req: RaftCmdRequest,
    mut err_resp: RaftCmdResponse,
    mut disk_full_opt: DiskFullOpt,
) -> bool {

    ...

    let policy = self.inspect(&req);
    let res = match policy {
        Ok(RequestPolicy::ReadLocal) | Ok(RequestPolicy::StaleRead) => {
            self.read_local(ctx, req, cb);
            return false;
        }
        Ok(RequestPolicy::ReadIndex) => return self.read_index(ctx, req, err_resp, cb),
        Ok(RequestPolicy::ProposeTransferLeader) => {
            return self.propose_transfer_leader(ctx, req, cb);
        }
        Ok(RequestPolicy::ProposeNormal) => {
            // For admin cmds, only region split/merge comes here.
            if req.has_admin_request() {
                disk_full_opt = DiskFullOpt::AllowedOnAlmostFull;
            }
            self.check_normal_proposal_with_disk_full_opt(ctx, disk_full_opt)
                .and_then(|_| self.propose_normal(ctx, req))
        }
        Ok(RequestPolicy::ProposeConfChange) => self.propose_conf_change(ctx, &req),
        Err(e) => Err(e),
    };
    fail_point!("after_propose");

    ...
}

fn read_index<T: Transport>(
    &mut self,
    poll_ctx: &mut PollContext<EK, ER, T>,
    mut req: RaftCmdRequest,
    mut err_resp: RaftCmdResponse,
    cb: Callback<EK::Snapshot>,
) -> bool {

    ...

    let mut read = ReadIndexRequest::with_command(id, req, cb, now);
    read.addition_request = request.map(Box::new);
    self.push_pending_read(read, self.is_leader());
    self.should_wake_up = true;

    ...

    true
}
```

那么什么条件满足后该 ReadIndexRequest 会被 pop 出队列并执行呢？

前面已经提到 ApplyBatchSystem 在应用一批日志之后首先会调用对应的 callback 尽快回复客户端，之后会发送一条 ApplyRes 的消息到 RaftBatchSystem，该消息和以上的 ReadIndex 请求一样被 PollHandler 在一次 loop 中被处理，并最终进入 `PeerFsmDelegate::handle_msgs` 函数的 `PeerMsg::ApplyRes { res }` 分支，接着其会调用 `PeerFsmDelegate::on_apply_res` 函数并进入 `store::peer::post_apply` 函数，在该函数中，ApplyRes 中携带的信息会被用来更新一些内存状态例如 `raft_group` 和 `cmd_epoch_checker`，当然，这些信息也会通过 `store::peer::post_pending_read_index_on_replica` 和 `self.pending_reads.pop_front()` 来释放某些满足条件的 ReadIndexRequest，对于每个 ReadIndexRequest ，此时可以通过 `store::peer::response_read` 函数来获取底层引擎的 Snapshot 并执行 callback 返回。

```rust
fn on_apply_res(&mut self, res: ApplyTaskRes<EK::Snapshot>) {
    fail_point!("on_apply_res", |_| {});
    match res {
        ApplyTaskRes::Apply(mut res) => {
            
            ...

            self.fsm.has_ready |= self.fsm.peer.post_apply(
                self.ctx,
                res.apply_state,
                res.applied_term,
                &res.metrics,
            );
        
            ...
        }
        ApplyTaskRes::Destroy {
            region_id,
            peer_id,
            merge_from_snapshot,
        } => {
            ...
        }
    }
}

pub fn post_apply<T>(
    &mut self,
    ctx: &mut PollContext<EK, ER, T>,
    apply_state: RaftApplyState,
    applied_term: u64,
    apply_metrics: &ApplyMetrics,
) -> bool {
    let mut has_ready = false;

    if self.is_handling_snapshot() {
        panic!("{} should not applying snapshot.", self.tag);
    }

    let applied_index = apply_state.get_applied_index();
    self.raft_group.advance_apply_to(applied_index);

    self.cmd_epoch_checker.advance_apply(
        applied_index,
        self.term(),
        self.raft_group.store().region(),
    );

    ...

    if !self.is_leader() {
        self.post_pending_read_index_on_replica(ctx)
    } else if self.ready_to_handle_read() {
        while let Some(mut read) = self.pending_reads.pop_front() {
            self.response_read(&mut read, ctx, false);
        }
    }
    self.pending_reads.gc();

    ...

    has_ready
}
```

综上，ReadIndexRequest 入队和出队的时机已经被介绍，那么 ReadIndex 的整体流程也基本介绍完整了。

通过本小节，希望您能够了解 KVGet 读请求的完整流程，并进而具备分析其他读请求全链路的能力。

## 总结

本篇博客介绍了 TiKV 中一条读请求的全链路流程。

希望本博客能够帮助对 TiKV 开发感兴趣的新同学尽快了解 TiKV 的 codebase。
