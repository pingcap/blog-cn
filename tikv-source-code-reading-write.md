---
title: TiKV 源码阅读三部曲（三）写流程
author: ['谭新宇']
date: 2022-11-09
summary: 本文是 TiKV 源码阅读三部曲第三篇，主要介绍了 TiKV 中一条写请求的全链路流程。
tags: ["TiKV 源码阅读"]
---

## 背景

[TiKV](https://github.com/tikv/tikv) 是一个支持事务的分布式 Key-Value 数据库，目前已经是 [CNCF 基金会](https://www.cncf.io/projects/) 的顶级项目。

作为一个新同学，需要一定的前期准备才能够有能力参与 TiKV 社区的代码开发，包括但不限于学习 Rust 语言，理解 TiKV 的原理和在前两者的基础上了解熟悉 TiKV 的源码。

笔者将结合[TiKV 官方源码解析文档](/blog/?tag=TiKV%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90) 系列文章，基于 **6.1 版本的源码**撰写三篇博客，分别介绍以下三个方面：
* [TiKV 源码阅读三部曲（一）重要模块](/blog/tikv-source-code-reading-module)：TiKV 的基本概念，TiKV 读写路径上的三个重要模块（KVService，Storage，RaftStore）和断点调试 TiKV 学习源码的方案
* [TiKV 源码阅读三部曲（二）读流程](/blog/tikv-source-code-reading-read)：TiKV 中一条读请求的全链路流程
* TiKV 源码阅读三部曲（三）写流程：TiKV 中一条写请求的全链路流程

希望此三篇博客能够帮助对 TiKV 开发感兴趣的新同学尽快了解 TiKV 的 codebase。

本文为第三篇博客，将主要介绍 TiKV 中一条写请求的全链路流程。

## 写流程

以下四篇博客由上到下分别介绍了 TiKV 3.x 版本 KVService，Storage 和 RaftStore 模块对于分布式事务请求的执行流程。
* [TiKV 源码解析系列文章（九）Service 层处理流程解析](/blog/tikv-source-code-reading-9)
* [TiKV 源码解析系列文章（十一）Storage - 事务控制层](/blog/tikv-source-code-reading-11)
* [TiKV 源码解析系列文章（十二）分布式事务](/blog/tikv-source-code-reading-12)
* [TiKV 源码解析系列文章（十八）Raft Propose 的 Commit 和 Apply 情景分析](/blog/tikv-source-code-reading-18)

本小节将在 TiKV 6.1 版本的基础上，以一条 PreWrite 请求为例，介绍当前版本的写请求全链路执行流程。

### KVService

在 KVService 层，通过 handle_request 和 txn_command_future 宏，PreWrite 接口的请求会直接被路由到 `Storage::sched_txn_command` 函数中。

```Rust
impl<T: RaftStoreRouter<E::Local> + 'static, E: Engine, L: LockManager, F: KvFormat> Tikv
    for Service<T, E, L, F>
{
    handle_request!(
        kv_prewrite,
        future_prewrite,
        PrewriteRequest,
        PrewriteResponse,
        has_time_detail
    );
} 

txn_command_future!(future_prewrite, PrewriteRequest, PrewriteResponse, (v, resp, tracker) {{
    if let Ok(v) = &v {
        resp.set_min_commit_ts(v.min_commit_ts.into_inner());
        resp.set_one_pc_commit_ts(v.one_pc_commit_ts.into_inner());
        GLOBAL_TRACKERS.with_tracker(tracker, |tracker| {
            tracker.write_scan_detail(resp.mut_exec_details_v2().mut_scan_detail_v2());
            tracker.write_write_detail(resp.mut_exec_details_v2().mut_write_detail());
        });
    }
    resp.set_errors(extract_key_errors(v.map(|v| v.locks)).into());
}});
```

### Storage

在 Storage 模块，其会将请求路由到 `Scheduler::run_cmd` 函数中，并进一步路由到 `Scheduler::schedule_command` 函数中。在 `schedule_command` 函数中，当前 command 连同 callback 等上下文会被保存到 task_slots 中，如果当前线程申请到了所有 latch 则会调用 execute 函数继续执行该 task，否则如前文所述，当前任务便会被阻塞在某些 latch 上等待其他线程去唤醒进而执行，当前线程会直接返回并执行其他的工作。

```rust
// The entry point of the storage scheduler. Not only transaction commands need
// to access keys serially.
pub fn sched_txn_command<T: StorageCallbackType>(
    &self,
    cmd: TypedCommand<T>,
    callback: Callback<T>,
) -> Result<()> {

    ...
    
    self.sched.run_cmd(cmd, T::callback(callback));

    Ok(())
}

pub(in crate::storage) fn run_cmd(&self, cmd: Command, callback: StorageCallback) {
    // write flow control
    if cmd.need_flow_control() && self.inner.too_busy(cmd.ctx().region_id) {
        SCHED_TOO_BUSY_COUNTER_VEC.get(cmd.tag()).inc();
        callback.execute(ProcessResult::Failed {
            err: StorageError::from(StorageErrorInner::SchedTooBusy),
        });
        return;
    }
    self.schedule_command(cmd, callback);
}

fn schedule_command(&self, cmd: Command, callback: StorageCallback) {
    let cid = self.inner.gen_id();
    let tracker = get_tls_tracker_token();
    debug!("received new command"; "cid" => cid, "cmd" => ?cmd, "tracker" => ?tracker);
    let tag = cmd.tag();
    let priority_tag = get_priority_tag(cmd.priority());
    SCHED_STAGE_COUNTER_VEC.get(tag).new.inc();
    SCHED_COMMANDS_PRI_COUNTER_VEC_STATIC
        .get(priority_tag)
        .inc();

    let mut task_slot = self.inner.get_task_slot(cid);
    let tctx = task_slot.entry(cid).or_insert_with(|| {
        self.inner
            .new_task_context(Task::new(cid, tracker, cmd), callback)
    });

    if self.inner.latches.acquire(&mut tctx.lock, cid) {
        fail_point!("txn_scheduler_acquire_success");
        tctx.on_schedule();
        let task = tctx.task.take().unwrap();
        drop(task_slot);
        self.execute(task);
        return;
    }
    let task = tctx.task.as_ref().unwrap();
    let deadline = task.cmd.deadline();
    let cmd_ctx = task.cmd.ctx().clone();
    self.fail_fast_or_check_deadline(cid, tag, cmd_ctx, deadline);
    fail_point!("txn_scheduler_acquire_fail");
}
```

在 execute 函数中，当前线程会生成一个异步任务 spawn 到另一个 worker 线程池中去，该任务主要包含以下两个步骤：
* 使用 `Self::with_tls_engine(|engine| Self::snapshot(engine, snap_ctx)).await` 获取 snapshot。此步骤与上文读流程中获取 snapshot 的步骤相同，可能通过 ReadLocal 也可能通过 ReadIndex 来获取引擎的 snapshot，此小节不在赘述
* 使用 `sched.process(snapshot, task).await` 基于获取到的 snapshot 和对应 task 去调用 `scheduler::process` 函数，进而被路由到 `scheduler::process_write` 函数中

```rust 
/// Executes the task in the sched pool.
fn execute(&self, mut task: Task) {
    set_tls_tracker_token(task.tracker);
    let sched = self.clone();
    self.get_sched_pool(task.cmd.priority())
        .pool
        .spawn(async move {
        
            ...

            // The program is currently in scheduler worker threads.
            // Safety: `self.inner.worker_pool` should ensure that a TLS engine exists.
            match unsafe { with_tls_engine(|engine: &E| kv::snapshot(engine, snap_ctx)) }.await
            {
                Ok(snapshot) => {
              
                    ...

                    sched.process(snapshot, task).await;
                }
                Err(err) => {
                    ...
                }
            }
        })
        .unwrap();
}

 /// Process the task in the current thread.
async fn process(self, snapshot: E::Snap, task: Task) {
    if self.check_task_deadline_exceeded(&task) {
        return;
    }

    let resource_tag = self.inner.resource_tag_factory.new_tag(task.cmd.ctx());
    async {
        
        ...

        if task.cmd.readonly() {
            self.process_read(snapshot, task, &mut statistics);
        } else {
            self.process_write(snapshot, task, &mut statistics).await;
        };
   
        ...
    }
    .in_resource_metering_tag(resource_tag)
    .await;
}
```

`scheduler::process_write` 函数是事务处理的关键函数，目前已经有近四百行，里面夹杂了很多新特性和新优化的复杂逻辑，其中最重要的逻辑有两个：
* 使用 `task.cmd.process_write(snapshot, context).map_err(StorageError::from)` 根据 snapshot 和 task 执行事务对应的语义：可以从 `Command::process_write` 函数看到不同的请求都有不同的实现，每种请求都可能根据 snapshot 去底层获取一些数据并尝试写入一些数据。有关 PreWrite 和其他请求的具体操作可以参照 [TiKV 源码解析系列文章（十二）分布式事务](/blog/tikv-source-code-reading-12)，此处不再赘述。需要注意的是，此时的写入仅仅缓存在了 WriteData 中，并没有对底层引擎进行实际修改。
* 使用 `engine.async_write_ext(&ctx, to_be_write, engine_cb, proposed_cb, committed_cb)` 将缓存的 WriteData 实际写入到 engine 层，对于 RaftKV 来说则是表示一次 propose，想要对这一批 WriteData commit 且 apply

```rust
async fn process_write(self, snapshot: E::Snap, task: Task, statistics: &mut Statistics) {
 
    ...

    let write_result = {
        let _guard = sample.observe_cpu();
        let context = WriteContext {
            lock_mgr: &self.inner.lock_mgr,
            concurrency_manager: self.inner.concurrency_manager.clone(),
            extra_op: task.extra_op,
            statistics,
            async_apply_prewrite: self.inner.enable_async_apply_prewrite,
        };
        let begin_instant = Instant::now();
        let res = unsafe {
            with_perf_context::<E, _, _>(tag, || {
                task.cmd
                    .process_write(snapshot, context)
                    .map_err(StorageError::from)
            })
        };
        SCHED_PROCESSING_READ_HISTOGRAM_STATIC
            .get(tag)
            .observe(begin_instant.saturating_elapsed_secs());
        res
    };

    ...

    // Safety: `self.sched_pool` ensures a TLS engine exists.
    unsafe {
        with_tls_engine(|engine: &E| {
            if let Err(e) =
                engine.async_write_ext(&ctx, to_be_write, engine_cb, proposed_cb, committed_cb)
            {
                SCHED_STAGE_COUNTER_VEC.get(tag).async_write_err.inc();

                info!("engine async_write failed"; "cid" => cid, "err" => ?e);
                scheduler.finish_with_err(cid, e);
            }
        })
    }
}

pub(crate) fn process_write<S: Snapshot, L: LockManager>(
    self,
    snapshot: S,
    context: WriteContext<'_, L>,
) -> Result<WriteResult> {
    match self {
        Command::Prewrite(t) => t.process_write(snapshot, context),
        Command::PrewritePessimistic(t) => t.process_write(snapshot, context),
        Command::AcquirePessimisticLock(t) => t.process_write(snapshot, context),
        Command::Commit(t) => t.process_write(snapshot, context),
        Command::Cleanup(t) => t.process_write(snapshot, context),
        Command::Rollback(t) => t.process_write(snapshot, context),
        Command::PessimisticRollback(t) => t.process_write(snapshot, context),
        Command::ResolveLock(t) => t.process_write(snapshot, context),
        Command::ResolveLockLite(t) => t.process_write(snapshot, context),
        Command::TxnHeartBeat(t) => t.process_write(snapshot, context),
        Command::CheckTxnStatus(t) => t.process_write(snapshot, context),
        Command::CheckSecondaryLocks(t) => t.process_write(snapshot, context),
        Command::Pause(t) => t.process_write(snapshot, context),
        Command::RawCompareAndSwap(t) => t.process_write(snapshot, context),
        Command::RawAtomicStore(t) => t.process_write(snapshot, context),
        _ => panic!("unsupported write command"),
    }
}

fn async_write_ext(
    &self,
    ctx: &Context,
    batch: WriteData,
    write_cb: Callback<()>,
    proposed_cb: Option<ExtCallback>,
    committed_cb: Option<ExtCallback>,
) -> kv::Result<()> {
    fail_point!("raftkv_async_write");
    if batch.modifies.is_empty() {
        return Err(KvError::from(KvErrorInner::EmptyRequest));
    }

    ASYNC_REQUESTS_COUNTER_VEC.write.all.inc();
    let begin_instant = Instant::now_coarse();

    self.exec_write_requests(
        ctx,
        batch,
        Box::new(move |res| match res {

            ...

        }),
        proposed_cb,
        committed_cb,
    )
    .map_err(|e| {
        let status_kind = get_status_kind_from_error(&e);
        ASYNC_REQUESTS_COUNTER_VEC.write.get(status_kind).inc();
        e.into()
    })
}
```

进入 `raftkv::async_write_ext` 函数后，其进而通过 `raftkv::exec_write_requests -> RaftStoreRouter::send_command` 的调用栈将 task 连带 callback 发送给 RaftBatchSystem 交由 RaftStore 模块处理。

```rust
fn exec_write_requests(
    &self,
    ctx: &Context,
    batch: WriteData,
    write_cb: Callback<CmdRes<E::Snapshot>>,
    proposed_cb: Option<ExtCallback>,
    committed_cb: Option<ExtCallback>,
) -> Result<()> {
    
    ...

    let cb = StoreCallback::write_ext(
        Box::new(move |resp| {
            write_cb(on_write_result(resp).map_err(Error::into));
        }),
        proposed_cb,
        committed_cb,
    );
    let extra_opts = RaftCmdExtraOpts {
        deadline: batch.deadline,
        disk_full_opt: batch.disk_full_opt,
    };
    self.router.send_command(cmd, cb, extra_opts)?;

    Ok(())
}

    /// Sends RaftCmdRequest to local store.
fn send_command(
    &self,
    req: RaftCmdRequest,
    cb: Callback<EK::Snapshot>,
    extra_opts: RaftCmdExtraOpts,
) -> RaftStoreResult<()> {
    send_command_impl::<EK, _>(self, req, cb, extra_opts)
}
```

### RaftStore

直接定位到 `RaftPoller` 的 `handle_normal` 函数。

与处理 ReadIndex 请求相似， `RaftPoller` 会首先尝试获取 `messages_per_tick` 次路由到该状态机的消息，接着调用 `PeerFsmDelegate::handle_msgs` 函数进行处理，

这里依然只列出了我们需要关注的几种消息类型：
* RaftMessage: 其他 Peer 发送过来 Raft 消息，包括心跳、日志、投票消息等。
* RaftCommand: 上层提出的 proposal，其中包含了需要通过 Raft 同步的操作，以及操作成功之后需要调用的 callback 函数。PreWrite 包装出的 RaftCommand 便是最正常的 proposal。
* ApplyRes: ApplyFsm 在将日志应用到状态机之后发送给 PeerFsm 的消息，用于在进行操作之后更新某些内存状态。

对于 PreWrite 请求，其会进入 `PeerMsg::RaftCommand(cmd)` 分支，进而以 `PeerFsmDelegate::propose_raft_command -> PeerFsmDelegate::propose_raft_command_internal -> Peer::propose -> Peer::propose_normal` 的调用链最终被 propose 到 raft-rs 的 RawNode 接口中，同时其 callback 会连带该请求的 logIndex 被 push 到该 Peer 的 `proposals` 中去。

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

     match res {
            Err(e) => {
                cmd_resp::bind_error(&mut err_resp, e);
                cb.invoke_with_response(err_resp);
                self.post_propose_fail(req_admin_cmd_type);
                false
            }
            Ok(Either::Right(idx)) => {
                if !cb.is_none() {
                    self.cmd_epoch_checker.attach_to_conflict_cmd(idx, cb);
                }
                self.post_propose_fail(req_admin_cmd_type);
                false
            }
            Ok(Either::Left(idx)) => {
                let has_applied_to_current_term = self.has_applied_to_current_term();
                if has_applied_to_current_term {
                    // After this peer has applied to current term and passed above checking
                    // including `cmd_epoch_checker`, we can safely guarantee
                    // that this proposal will be committed if there is no abnormal leader transfer
                    // in the near future. Thus proposed callback can be called.
                    cb.invoke_proposed();
                }
                if is_urgent {
                    self.last_urgent_proposal_idx = idx;
                    // Eager flush to make urgent proposal be applied on all nodes as soon as
                    // possible.
                    self.raft_group.skip_bcast_commit(false);
                }
                self.should_wake_up = true;
                let p = Proposal {
                    is_conf_change: req_admin_cmd_type == Some(AdminCmdType::ChangePeer)
                        || req_admin_cmd_type == Some(AdminCmdType::ChangePeerV2),
                    index: idx,
                    term: self.term(),
                    cb,
                    propose_time: None,
                    must_pass_epoch_check: has_applied_to_current_term,
                };
                if let Some(cmd_type) = req_admin_cmd_type {
                    self.cmd_epoch_checker
                        .post_propose(cmd_type, idx, self.term());
                }
                self.post_propose(ctx, p);
                true
            }
        }
}
```

在调用完 `PeerFsmDelegate::handle_msgs` 处理完消息后，会再调用 `PeerFsmDelegate::collect_ready()` 函数，进而进入 `Peer::handle_raft_ready_append` 函数。在该函数中会收集 normal 状态机的一次 ready，接着对需要持久化的未提交日志进行持久化（延后攒批），需要发送的消息进行异步发送，需要应用的已提交日志发送给 ApplyBatchSystem。

在三副本情况下，该 PreWrite 请求会存在于本次 ready 需要持久化的日志和需要发往其他两个 peer 的 message 中，对于 message，一旦收到就会 spawn 给 Transport 让其异步发送，对于持久化，在不开启 async-io 的情况下，数据会被暂存到内存中在当前 loop 结尾的 end 函数中实际写入到底层引擎中去。

```rust
/// Collect ready if any.
///
/// Returns false is no readiness is generated.
pub fn collect_ready(&mut self) -> bool {
    ...

    let res = self.fsm.peer.handle_raft_ready_append(self.ctx);

    ...

}
pub fn handle_raft_ready_append<T: Transport>(
    &mut self,
    ctx: &mut PollContext<EK, ER, T>,
) -> Option<ReadyResult> {

    ...

    if !self.raft_group.has_ready() {
        fail_point!("before_no_ready_gen_snap_task", |_| None);
        // Generating snapshot task won't set ready for raft group.
        if let Some(gen_task) = self.mut_store().take_gen_snap_task() {
            self.pending_request_snapshot_count
                .fetch_add(1, Ordering::SeqCst);
            ctx.apply_router
                .schedule_task(self.region_id, ApplyTask::Snapshot(gen_task));
        }
        return None;
    }

    ...
    
    let mut ready = self.raft_group.ready();

    ...

    if !ready.must_sync() {
        // If this ready need not to sync, the term, vote must not be changed,
        // entries and snapshot must be empty.
        if let Some(hs) = ready.hs() {
            assert_eq!(hs.get_term(), self.get_store().hard_state().get_term());
            assert_eq!(hs.get_vote(), self.get_store().hard_state().get_vote());
        }
        assert!(ready.entries().is_empty());
        assert!(ready.snapshot().is_empty());
    }

    self.on_role_changed(ctx, &ready);

    if let Some(hs) = ready.hs() {
        let pre_commit_index = self.get_store().commit_index();
        assert!(hs.get_commit() >= pre_commit_index);
        if self.is_leader() {
            self.on_leader_commit_idx_changed(pre_commit_index, hs.get_commit());
        }
    }

    if !ready.messages().is_empty() {
        assert!(self.is_leader());
        let raft_msgs = self.build_raft_messages(ctx, ready.take_messages());
        self.send_raft_messages(ctx, raft_msgs);
    }

    self.apply_reads(ctx, &ready);

    if !ready.committed_entries().is_empty() {
        self.handle_raft_committed_entries(ctx, ready.take_committed_entries());
    }

    ...

    let ready_number = ready.number();
    let persisted_msgs = ready.take_persisted_messages();
    let mut has_write_ready = false;
    match &res {
        HandleReadyResult::SendIoTask | HandleReadyResult::Snapshot { .. } => {
            if !persisted_msgs.is_empty() {
                task.messages = self.build_raft_messages(ctx, persisted_msgs);
            }

            if !trackers.is_empty() {
                task.trackers = trackers;
            }

            if let Some(write_worker) = &mut ctx.sync_write_worker {
                write_worker.handle_write_task(task);

                assert_eq!(self.unpersisted_ready, None);
                self.unpersisted_ready = Some(ready);
                has_write_ready = true;
            } else {
                self.write_router.send_write_msg(
                    ctx,
                    self.unpersisted_readies.back().map(|r| r.number),
                    WriteMsg::WriteTask(task),
                );

                self.unpersisted_readies.push_back(UnpersistedReady {
                    number: ready_number,
                    max_empty_number: ready_number,
                    raft_msgs: vec![],
                });

                self.raft_group.advance_append_async(ready);
            }
        }
        HandleReadyResult::NoIoTask => {
            if let Some(last) = self.unpersisted_readies.back_mut() {
                // Attach to the last unpersisted ready so that it can be considered to be
                // persisted with the last ready at the same time.
                if ready_number <= last.max_empty_number {
                    panic!(
                        "{} ready number is not monotonically increaing, {} <= {}",
                        self.tag, ready_number, last.max_empty_number
                    );
                }
                last.max_empty_number = ready_number;

                if !persisted_msgs.is_empty() {
                    self.unpersisted_message_count += persisted_msgs.capacity();
                    last.raft_msgs.push(persisted_msgs);
                }
            } else {
                // If this ready don't need to be persisted and there is no previous unpersisted
                // ready, we can safely consider it is persisted so the persisted msgs can be
                // sent immediately.
                self.persisted_number = ready_number;

                if !persisted_msgs.is_empty() {
                    fail_point!("raft_before_follower_send");
                    let msgs = self.build_raft_messages(ctx, persisted_msgs);
                    self.send_raft_messages(ctx, msgs);
                }

                // The commit index and messages of light ready should be empty because no data
                // needs to be persisted.
                let mut light_rd = self.raft_group.advance_append(ready);

                self.add_light_ready_metric(&light_rd, &mut ctx.raft_metrics);

                if let Some(idx) = light_rd.commit_index() {
                    panic!(
                        "{} advance ready that has no io task but commit index is changed to {}",
                        self.tag, idx
                    );
                }
                if !light_rd.messages().is_empty() {
                    panic!(
                        "{} advance ready that has no io task but message is not empty {:?}",
                        self.tag,
                        light_rd.messages()
                    );
                }
                // The committed entries may not be empty when the size is too large to
                // be fetched in the previous ready.
                if !light_rd.committed_entries().is_empty() {
                    self.handle_raft_committed_entries(ctx, light_rd.take_committed_entries());
                }
            }
        }
    }

    ...
} 
```

等到任何一个 follower 返回确认后，该 response 会被路由到 RaftBatchSystem，PollHandler 在接下来的一次 loop 中对其进行处理，该请求会被路由到 `PeerFsmDelegate::handle_msgs` 函数的 `PeerMsg::RaftMessage(msg)` 分支中，进而调用 step 函数交给 raft-rs 状态机进行处理。

由于此时已经满足了 quorum 的写入，raft-rs 会将该 PreWrite 请求对应的 raftlog 进行提交并在下一次被获取 ready 时返回，在本轮 loop 的 `PeerFsmDelegate::collect_ready()` 函数及 `Peer::handle_raft_ready_append` 函数中，会调用 `self.handle_raft_committed_entries(ctx, ready.take_committed_entries())` 函数。在该函数中，其会根据已提交日志从 Peer 的 `proposals` 中获取到对应的 callback，连带这一批所有的已提交日志构建一个 Apply Task 通过 apply_router 发送给 ApplyBatchSystem。

```rust
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

fn handle_raft_committed_entries<T>(
    &mut self,
    ctx: &mut PollContext<EK, ER, T>,
    committed_entries: Vec<Entry>,
) {
    if committed_entries.is_empty() {
        return;
    }

    ...

    if let Some(last_entry) = committed_entries.last() {
        self.last_applying_idx = last_entry.get_index();
        if self.last_applying_idx >= self.last_urgent_proposal_idx {
            // Urgent requests are flushed, make it lazy again.
            self.raft_group.skip_bcast_commit(true);
            self.last_urgent_proposal_idx = u64::MAX;
        }
        let cbs = if !self.proposals.is_empty() {
            let current_term = self.term();
            let cbs = committed_entries
                .iter()
                .filter_map(|e| {
                    self.proposals
                        .find_proposal(e.get_term(), e.get_index(), current_term)
                })
                .map(|mut p| {
                    if p.must_pass_epoch_check {
                        // In this case the apply can be guaranteed to be successful. Invoke the
                        // on_committed callback if necessary.
                        p.cb.invoke_committed();
                    }
                    p
                })
                .collect();
            self.proposals.gc();
            cbs
        } else {
            vec![]
        };
        // Note that the `commit_index` and `commit_term` here may be used to
        // forward the commit index. So it must be less than or equal to persist
        // index.
        let commit_index = cmp::min(
            self.raft_group.raft.raft_log.committed,
            self.raft_group.raft.raft_log.persisted,
        );
        let commit_term = self.get_store().term(commit_index).unwrap();
        let mut apply = Apply::new(
            self.peer_id(),
            self.region_id,
            self.term(),
            commit_index,
            commit_term,
            committed_entries,
            cbs,
            self.region_buckets.as_ref().map(|b| b.meta.clone()),
        );
        apply.on_schedule(&ctx.raft_metrics);
        self.mut_store()
            .trace_cached_entries(apply.entries[0].clone());
        if needs_evict_entry_cache(ctx.cfg.evict_cache_on_memory_ratio) {
            // Compact all cached entries instead of half evict.
            self.mut_store().evict_entry_cache(false);
        }
        ctx.apply_router
            .schedule_task(self.region_id, ApplyTask::apply(apply));
    }
    fail_point!("after_send_to_apply_1003", self.peer_id() == 1003, |_| {});
}
```

此时直接定位到 `ApplyPoller` 的 `handle_normal` 函数，可以看到，`ApplyPoller` 也会首先尝试获取 `messages_per_tick` 次路由到该状态机的消息，接着调用 `ApplyFSM::handle_tasks` 函数进行处理。然后其会经历 `ApplyFSM::handle_apply ->  ApplyDelegate::handle_raft_committed_entries` 的调用链来到 `ApplyDelegate::handle_raft_entry_normal` 函数中，在该函数中，会尝试将调用 `ApplyDelegate::process_raft_cmd` 函数来将本次写入缓存到 `kv_write_batch` 中，值得一提的是，在写入缓存之前会首先判断是否能够进行一次提交，如果可以则需要在写入缓存之前将这一批日志提交到底层引擎。

```rust
fn handle_normal(&mut self, normal: &mut impl DerefMut<Target = ApplyFsm<EK>>) -> HandleResult {

    ...
    
    while self.msg_buf.len() < self.messages_per_tick {
        match normal.receiver.try_recv() {
            Ok(msg) => self.msg_buf.push(msg),
            Err(TryRecvError::Empty) => {
                handle_result = HandleResult::stop_at(0, false);
                break;
            }
            Err(TryRecvError::Disconnected) => {
                normal.delegate.stopped = true;
                handle_result = HandleResult::stop_at(0, false);
                break;
            }
        }
    }

    normal.handle_tasks(&mut self.apply_ctx, &mut self.msg_buf);

    if normal.delegate.wait_merge_state.is_some() {
        // Check it again immediately as catching up logs can be very fast.
        handle_result = HandleResult::stop_at(0, false);
    } else if normal.delegate.yield_state.is_some() {
        // Let it continue to run next time.
        handle_result = HandleResult::KeepProcessing;
    }
    handle_result
}

fn handle_raft_entry_normal(
    &mut self,
    apply_ctx: &mut ApplyContext<EK>,
    entry: &Entry,
) -> ApplyResult<EK::Snapshot> {
    fail_point!(
        "yield_apply_first_region",
        self.region.get_start_key().is_empty() && !self.region.get_end_key().is_empty(),
        |_| ApplyResult::Yield
    );

    let index = entry.get_index();
    let term = entry.get_term();
    let data = entry.get_data();

    if !data.is_empty() {
        let cmd = util::parse_data_at(data, index, &self.tag);

        if apply_ctx.yield_high_latency_operation && has_high_latency_operation(&cmd) {
            self.priority = Priority::Low;
        }
        let mut has_unflushed_data =
            self.last_flush_applied_index != self.apply_state.get_applied_index();
        if has_unflushed_data && should_write_to_engine(&cmd)
            || apply_ctx.kv_wb().should_write_to_engine()
        {
            apply_ctx.commit(self);
            if let Some(start) = self.handle_start.as_ref() {
                if start.saturating_elapsed() >= apply_ctx.yield_duration {
                    return ApplyResult::Yield;
                }
            }
            has_unflushed_data = false;
        }
        if self.priority != apply_ctx.priority {
            if has_unflushed_data {
                apply_ctx.commit(self);
            }
            return ApplyResult::Yield;
        }

        return self.process_raft_cmd(apply_ctx, index, term, cmd);
    }

    ...
}
```

那么为什么不像 RaftBatchSystem 一样在 end 函数中统一进行攒批提交呢？原因是此时只要攒够一定的大小不对底层引擎造成过大的负载就可以快速提交并返回客户端了，等到最后再去处理只会增加写入延时而没有太大的收益。

让我们阅读一下提交 batch 的逻辑，其会经由 `ApplyContext::commit -> ApplyContext::commit_opt` 的调用链来到 ` ApplyContext::write_to_db` 函数，在该函数中，会调用 `self.kv_wb_mut().write_opt(&write_opts)` 函数将该 `WriteBatch` 提交到底层引擎，接着在最后调用 `cb.invoke_with_response(resp)` 来执行 callback 尽快返回客户端。 

```rust
/// Commits all changes have done for delegate. `persistent` indicates
/// whether write the changes into rocksdb.
///
/// This call is valid only when it's between a `prepare_for` and
/// `finish_for`.
pub fn commit(&mut self, delegate: &mut ApplyDelegate<EK>) {
    if delegate.last_flush_applied_index < delegate.apply_state.get_applied_index() {
        delegate.write_apply_state(self.kv_wb_mut());
    }
    self.commit_opt(delegate, true);
}
fn commit_opt(&mut self, delegate: &mut ApplyDelegate<EK>, persistent: bool) {
    delegate.update_metrics(self);
    if persistent {
        self.write_to_db();
        self.prepare_for(delegate);
        delegate.last_flush_applied_index = delegate.apply_state.get_applied_index()
    }
    self.kv_wb_last_bytes = self.kv_wb().data_size() as u64;
    self.kv_wb_last_keys = self.kv_wb().count() as u64;
}

/// Writes all the changes into RocksDB.
/// If it returns true, all pending writes are persisted in engines.
pub fn write_to_db(&mut self) -> bool {
    let need_sync = self.sync_log_hint;
    // There may be put and delete requests after ingest request in the same fsm.
    // To guarantee the correct order, we must ingest the pending_sst first, and
    // then persist the kv write batch to engine.
    if !self.pending_ssts.is_empty() {
        let tag = self.tag.clone();
        self.importer
            .ingest(&self.pending_ssts, &self.engine)
            .unwrap_or_else(|e| {
                panic!(
                    "{} failed to ingest ssts {:?}: {:?}",
                    tag, self.pending_ssts, e
                );
            });
        self.pending_ssts = vec![];
    }
    if !self.kv_wb_mut().is_empty() {
        self.perf_context.start_observe();
        let mut write_opts = engine_traits::WriteOptions::new();
        write_opts.set_sync(need_sync);
        self.kv_wb_mut().write_opt(&write_opts).unwrap_or_else(|e| {
            panic!("failed to write to engine: {:?}", e);
        });
        let trackers: Vec<_> = self
            .applied_batch
            .cb_batch
            .iter()
            .flat_map(|(cb, _)| cb.write_trackers())
            .flat_map(|trackers| trackers.iter().map(|t| t.as_tracker_token()))
            .flatten()
            .collect();
        self.perf_context.report_metrics(&trackers);
        self.sync_log_hint = false;
        let data_size = self.kv_wb().data_size();
        if data_size > APPLY_WB_SHRINK_SIZE {
            // Control the memory usage for the WriteBatch.
            self.kv_wb = self.engine.write_batch_with_cap(DEFAULT_APPLY_WB_SIZE);
        } else {
            // Clear data, reuse the WriteBatch, this can reduce memory allocations and
            // deallocations.
            self.kv_wb_mut().clear();
        }
        self.kv_wb_last_bytes = 0;
        self.kv_wb_last_keys = 0;
    }
    if !self.delete_ssts.is_empty() {
        let tag = self.tag.clone();
        for sst in self.delete_ssts.drain(..) {
            self.importer.delete(&sst.meta).unwrap_or_else(|e| {
                panic!("{} cleanup ingested file {:?}: {:?}", tag, sst, e);
            });
        }
    }
    // Take the applied commands and their callback
    let ApplyCallbackBatch {
        cmd_batch,
        batch_max_level,
        mut cb_batch,
    } = mem::replace(&mut self.applied_batch, ApplyCallbackBatch::new());
    // Call it before invoking callback for preventing Commit is executed before
    // Prewrite is observed.
    self.host
        .on_flush_applied_cmd_batch(batch_max_level, cmd_batch, &self.engine);
    // Invoke callbacks
    let now = std::time::Instant::now();
    for (cb, resp) in cb_batch.drain(..) {
        for tracker in cb.write_trackers().iter().flat_map(|v| *v) {
            tracker.observe(now, &self.apply_time, |t| &mut t.metrics.apply_time_nanos);
        }
        cb.invoke_with_response(resp);
    }
    self.apply_time.flush();
    self.apply_wait.flush();
    need_sync
}
```

在 `ApplyPoller` 一轮 loop 结尾的 end 函数中，其会调用 `ApplyContext::flush` 函数，进而通过 `self.notifier.notify(apply_res)` 将 ApplyRes 重新发送到 RaftBatchSystem 中去，进而更新某些内存结构，此处不再赘述。

```rust
fn end(&mut self, fsms: &mut [Option<impl DerefMut<Target = ApplyFsm<EK>>>]) {
    self.apply_ctx.flush();
    for fsm in fsms.iter_mut().flatten() {
        fsm.delegate.last_flush_applied_index = fsm.delegate.apply_state.get_applied_index();
        fsm.delegate.update_memory_trace(&mut self.trace_event);
    }
    MEMTRACE_APPLYS.trace(mem::take(&mut self.trace_event));
}

    /// Flush all pending writes to engines.
/// If it returns true, all pending writes are persisted in engines.
pub fn flush(&mut self) -> bool {
    // TODO: this check is too hacky, need to be more verbose and less buggy.
    let t = match self.timer.take() {
        Some(t) => t,
        None => return false,
    };

    // Write to engine
    // raftstore.sync-log = true means we need prevent data loss when power failure.
    // take raft log gc for example, we write kv WAL first, then write raft WAL,
    // if power failure happen, raft WAL may synced to disk, but kv WAL may not.
    // so we use sync-log flag here.
    let is_synced = self.write_to_db();

    if !self.apply_res.is_empty() {
        fail_point!("before_nofity_apply_res");
        let apply_res = mem::take(&mut self.apply_res);
        self.notifier.notify(apply_res);
    }

    let elapsed = t.saturating_elapsed();
    STORE_APPLY_LOG_HISTOGRAM.observe(duration_to_sec(elapsed) as f64);
    for mut inspector in std::mem::take(&mut self.pending_latency_inspect) {
        inspector.record_apply_process(elapsed);
        inspector.finish();
    }

    slow_log!(
        elapsed,
        "{} handle ready {} committed entries",
        self.tag,
        self.committed_count
    );
    self.committed_count = 0;
    is_synced
}
```

通过本小节，希望您能够了解 PreWrite 请求的完整流程，并进而具备分析其他写请求全链路的能力。

## 总结

本篇博客介绍了 TiKV 中一条写请求的全链路流程。

希望本博客能够帮助对 TiKV 开发感兴趣的新同学尽快了解 TiKV 的 codebase。
