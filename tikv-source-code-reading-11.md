---
title: TiKV 源码解析系列文章（十一）Storage - 事务控制层
author: ['张金鹏']
date: 2019-07-29
summary: 本文将为大家介绍 TiKV 源码中的 Storage 模块，它位于 Service 与底层 KV 存储引擎之间，主要负责事务的并发控制。TiKV 端事务相关的实现都在 Storage 模块中。
tags: ['TiKV 源码解析','社区']
---


## 背景知识

TiKV 是一个强一致的支持事务的分布式 KV 存储。TiKV 通过 raft 来保证多副本之间的强一致，事务这块 TiKV 参考了 Google 的 [Percolator 事务模型](https://ai.google/research/pubs/pub36726)，并进行了一些优化。

当 TiKV 的 Service 层收到请求之后，会根据请求的类型把这些请求转发到不同的模块进行处理。对于从 TiDB 下推的读请求，比如 sum，avg 操作，会转发到 Coprocessor 模块进行处理，对于 KV 请求会直接转发到 Storage 进行处理。

KV 操作根据功能可以被划分为 Raw KV 操作以及 Txn KV 操作两大类。Raw KV 操作包括 raw put、raw get、raw delete、raw batch get、raw batch put、raw batch delete、raw scan 等普通 KV 操作。 Txn KV 操作是为了实现事务机制而设计的一系列操作，如 prewrite 和 commit 分别对应于 2PC 中的 prepare 和 commit 阶段的操作。

**本文将为大家介绍 TiKV 源码中的 Storage 模块，它位于 Service 与底层 KV 存储引擎之间，主要负责事务的并发控制。TiKV 端事务相关的实现都在 Storage 模块中。**

## 源码解析

接下来我们将从 Engine、Latches、Scheduler 和 MVCC 等几个方面来讲解 Storage 相关的源码。

### 1. Engine trait

TiKV 把底层 KV 存储引擎抽象成一个 Engine trait（trait 类似其他语言的 interface），定义见 `storage/kv/mod.rs`。Engint trait 主要提供了读和写两个接口，分别为 `async_snapshot` 和 `async_write`。调用者把要写的内容交给 `async_write`，`async_write` 通过回调的方式告诉调用者写操作成功完成了或者遇到错误了。同样的，`async_snapshot` 通过回调的方式把数据库的快照返回给调用者，供调用者读，或者把遇到的错误返回给调用者。

```rust
pub trait Engine: Send + Clone + 'static {
    type Snap: Snapshot;
    fn async_write(&self, ctx: &Contect, batch: Vec<Modify>, callback: Callback<()>) -> Result<()>;
    fn async_snapshot(&self, ctx: &Context, callback: Callback<Self::Snap>) -> Result<()>;
}
```

只要实现了以上两个接口，都可以作为 TiKV 的底层 KV 存储引擎。在 3.0 版本中，TiKV 支持了三种不同的 KV 存储引擎，包括单机 RocksDB 引擎、内存 B 树引擎和 RaftKV 引擎，分别位于 `storage/kv` 文件夹下面的 `rocksdb_engine.rs`、`btree_engine.rs` 和 `raftkv.rs`。其中单机 RocksDB 引擎和内存红黑树引擎主要用于单元测试和分层 benchmark，TiKV 真正使用的是 RaftKV 引擎。当调用 RaftKV 的 `async_write` 进行写入操作时，如果 `async_write` 通过回调方式成功返回了，说明写入操作已经通过 raft 复制给了大多数副本，并且在 leader 节点（调用者所在 TiKV）完成写入了，后续 leader 节点上的读就能够看到之前写入的内容。

### 2. Raw KV 执行流程

Raw KV 系列接口是绕过事务直接操纵底层数据的接口，没有事务控制，比较简单，所以在介绍更复杂的事务 KV 的执行流程前，我们先介绍 Raw KV 的执行流程。

#### Raw put

raw put 操作不需要 Storage 模块做额外的工作，直接把要写的内容通过 engine 的 `async_write` 接口发送给底层的 KV 存储引擎就好了。调用堆栈为 `service/kv.rs: raw_put` -> `storage/mod.rs: async_raw_put`。

```rust
impl<E: Engine> Storage<E> {
    pub fn async_raw_put(
        &self,
        ctx: Context,
        cf: String,
        key: Vec<u8>,
        value: Vec<u8>,
        callback: Callback<()>,
    ) -> Result<()> {
        // Omit some limit checks about key and value here...
        self.engine.async_write(
            &ctx,
            vec![Modify::Put(
                Self::rawkv_cf(&cf),
                Key::from_encoded(key),
                value,
            )],
            Box::new(|(_, res)| callback(res.map_err(Error::from))),
        )?;
        Ok(())
    }
}
```

#### Raw get

同样的，raw get 只需要调用 engine 的 `async_snapshot` 拿到数据库快照，然后直接读取就可以了。当然对于 RaftKV 引擎，`async_snapshot` 在返回数据库快照之前会做一些检查工作，比如会检查当前访问的副本是否是 leader（3.0.0 版本只支持从 leader 进行读操作，follower read 目前仍然在开发中），另外也会检查请求中携带的 region 版本信息是否足够新。

### 3. Latches

在事务模式下，为了防止多个请求同时对同一个 key 进行写操作，请求在写这个 key 之前必须先获取这个 key 的内存锁。为了和事务中的锁进行区分，我们称这个内存锁为 latch，对应的是 `storage/txn/latch.rs` 文件中的 Latch 结构体。每个 Latch 内部包含一个等待队列，没有拿到 latch 的请求按先后顺序插入到等待队列中，队首的请求被认为拿到了该 latch。

```rust
#[derive(Clone)]
struct Latch {
    pub waiting: VecDeque<u64>,
}
```

Latches 是一个包含多个 Latch 的结构体，内部包含一个固定长度的 Vector，Vector 的每个 slot 对应一个 Latch。默认配置下 Latches 内部 Vector 的长度为 2048000。每个 TiKV 有且仅有一个 Latches 实例，位于 `Storage.Scheduler` 中。

```rust
pub struct Latches {
    slots: Vec<Mutex<Latch>>,
    size: usize,
}
```

Latches 的 `gen_lock` 接口用于计算写入请求执行前所需要获取的所有 latch。`gen_lock` 通过计算所有 key 的 hash，然后用这些 hash 对 Vector 的长度进行取模得到多个 slots，对这些 slots 经过排序去重得到该命令需要的所有 latch。这个过程中的排序是为了保证获取 latch 的顺序性防止出现死锁情况。

```rust
impl Latches {
    pub fn gen_lock<H: Hash>(&self, keys: &[H]) -> Lock {
        // prevent from deadlock, so we sort and deduplicate the index.
        let mut slots: Vec<usize> = keys.iter().map(|x|
        self.calc_slot(x)).collect();
        slots.sort();
        slots.dedup();
        Lock::new(slots)
    }
}
```

### 4. Storage 和事务调度器 Scheduler

#### Storage

Storage 定义在 `storage/mod.rs` 文件中，下面我们介绍下 Storage 几个重要的成员：

`engine`：代表的是底层的 KV 存储引擎。

`sched`：事务调度器，负责并发事务请求的调度工作。

`read_pool`：读取线程池，所有只读 KV 请求，包括事务的非事务的，如 raw get、txn kv get 等最终都会在这个线程池内执行。由于只读请求不需要获取 latches，所以为其分配一个独立的线程池直接执行，而不是与非只读事务共用事务调度器。

`gc_worker`：从 3.0 版本开始，TiKV 支持分布式 GC，每个 TiKV 有一个 `gc_worker` 线程负责定期从 PD 更新 safepoint，然后进行 GC 工作。

`pessimistic_txn_enabled`： 另外 3.0 版本也支持悲观事务，`pessimistic_txn_enabled` 为 true 表示 TiKV 以支持悲观事务的模式启动，关于悲观事务后续会有一篇源码阅读文章专门介绍，这里我们先跳过。

```rust
pub struct Storage<E: Engine> {
    engine: E,
    sched: Scheduler<E>,
    read_pool: ReadPool,
    gc_worker: GCWorker<E>,
    pessimistic_txn_enabled: bool,
    // Other fields...
}
```

对于只读请求，包括 txn get 和 txn scan，Storage 调用 engine 的 `async_snapshot` 获取数据库快照之后交给 `read_pool` 线程池进行处理。写入请求，包括 prewrite、commit、rollback 等，直接交给 Scheduler 进行处理。Scheduler 的定义在 `storage/txn/scheduler.rs` 中。

#### Scheduler

```rust
pub struct Scheduler<E: Engine> {
    engine: Option<E>,
    inner: Arc<SchedulerInner>,
}

struct SchedulerInner {
    id_alloc: AtomicU64,
    task_contexts: Vec<Mutex<HashMap<u64, TaskContext>>>,
    lathes: Latches,
    sched_pending_write_threshold: usize,
    worker_pool: SchedPool,
    high_priority_pool: SchedPool,
    // Some other fields...
}
```

接下来简单介绍下 Scheduler 几个重要的成员：

`id_alloc`：到达 Scheduler 的请求都会被分配一个唯一的 command id。

`latches`：写请求到达 Scheduler 之后会尝试获取所需要的 latch，如果暂时获取不到所需要的 latch，其对应的 command id 会被插入到 latch 的 waiting list 里，当前面的请求执行结束后会唤醒 waiting list 里的请求继续执行，这部分逻辑我们将会在下一节 prewrite 请求在 scheduler 中的执行流程中介绍。

`task_contexts`：用于存储 Scheduler 中所有请求的上下文，比如暂时未能获取所需 latch 的请求都会被暂存在 `task_contexts` 中。

`sched_pending_write_threshold`：用于统计 Scheduler 内所有写入请求的写入流量，可以通过该指标对 Scheduler 的写入操作进行流控。

`worker_pool`，`high_priority_pool`：两个线程池，写请求在调用 engine 的 async_write 之前需要进行事务约束的检验工作，这些工作都是在这个两个线程池中执行的。

##### prewrite 请求在 Scheduler 中的执行流程

下面我们以 prewrite 请求为例子来讲解下写请求在 Scheduler 中是如何处理的：

1）Scheduler 收到 prewrite 请求的时候首先会进行流控判断，如果 Scheduler 里的请求过多，会直接返回 `SchedTooBusy` 错误，提示等一会再发送，否则进入下一步。

2）接着会尝试获取所需要的 latch，如果获取 latch 成功那么直接进入下一步。如果获取 latch 失败，说明有其他请求占住了 latch，这种情况说明其他请求可能也正在对相同的 key 进行操作，那么当前 prewrite 请求会被暂时挂起来，请求的上下文会暂存在 Scheduler 的 `task_contexts` 里面。当前面的请求执行结束之后会将该 prewrite 请求重新唤醒继续执行。

```rust
impl<E: Engine> Scheduler<E> {
    fn try_to_wake_up(&self, cid: u64) {
        if self.inner.acquire_lock(cid) {
            self.get_snapshot(cid);
        }
    }
    fn release_lock(&self, lock: &Lock, cid: u64) {
        let wakeup_list = self.inner.latches.release(lock, cid);
        for wcid in wakeup_list {
            self.try_to_wake_up(wcid);
        }
    }
}
```

3）获取 latch 成功之后会调用 Scheduler 的 `get_snapshot` 接口从 engine 获取数据库的快照。`get_snapshot` 内部实际上就是调用 engine 的 `async_snapshot` 接口。然后把 prewrite 请求以及刚刚获取到的数据库快照交给 `worker_pool` 进行处理。如果该 prewrite 请求优先级字段是 `high` 就会被分发到 `high_priority_pool` 进行处理。`high_priority_pool` 是为了那些高优先级请求而设计的，比如 TiDB 系统内部的一些请求要求 TiKV 快速返回，不能由于 `worker_pool` 繁忙而被卡住。需要注意的是，目前 `high_priority_pool` 与 `worker_pool` 仅仅是语义上不同的两个线程池，它们内部具有相同的操作系统调度优先级。

4）`worker_pool` 收到 prewrite 请求之后，主要工作是从拿到的数据库快照里确认当前 prewrite 请求是否能够执行，比如是否已经有更大 ts 的事务已经对数据进行了修改，具体的细节可以参考 [Percolator 论文](https://ai.google/research/pubs/pub36726)，或者参考我们的官方博客 [《TiKV 事务模型概览》](https://pingcap.com/blog-cn/tidb-transaction-model/)。当判断 prewrite 是可以执行的，会调用 engine 的 `async_write` 接口执行真正的写入操作。这部分的具体的代码见 `storage/txn/process.rs` 中的 `process_write_impl` 函数。

5）当 `async_write` 执行成功或失败之后，会调用 Scheduler 的 `release_lock` 函数来释放 latch 并且唤醒等待在这些 latch 上的请求继续执行。

### 5. MVCC

TiKV MVCC 相关的代码位于 `storage/mvcc` 文件夹下，强烈建议大家在阅读这部分代码之前先阅读 [Percolator 论文](https://ai.google/research/pubs/pub36726)，或者我们的官方博客 [《TiKV 事务模型概览》](https://pingcap.com/blog-cn/tidb-transaction-model/)。

MVCC 下面有两个比较关键的结构体，分别为 `MvccReader` 和 `MvccTxn`。`MvccReader` 位于 `storage/mvcc/reader/reader.rs` 文件中，它主要提供读功能，将多版本的处理细节隐藏在内部。比如 `MvccReader` 的 `get` 接口，传入需要读的 key 以及 ts，返回这个 ts 可以看到的版本或者返回 `key is lock` 错误等。

```rust
impl<S: Snapshot> MvccReader<S> {
    pub fn get(&mut self, key: &Key, mut ts: u64) -> Result<Option<Value>>;
}
```

`MvccTxn` 位于 `storage/mvcc/txn.rs` 文件中，它主要提供写之前的事务约束检验功能，上一节 prewrite 请求的处理流程中第四步就是通过调用 `MvccTxn` 的 prewrite 接口来进行的事务约束检验。

## 小结

TiKV 端事务相关的实现都位于 Storage 模块中，该文带大家简单概览了下这部分几个关键的点，想了解更多细节的读者可以自行阅读这部分的源码（code talks XD）。另外从 3.0 版本开始，TiDB 和 TiKV 支持悲观事务，TiKV 端对应的代码主要位于 `storage/lock_manager` 以及上面提到的 MVCC 模块中。
