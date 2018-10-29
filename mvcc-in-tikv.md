---
title: TiKV 的 MVCC（Multi-Version Concurrency Control）机制
author: ['ivan yang']
date: 2016-11-22
summary: 事务隔离在数据库系统中有着非常重要的作用，因为对于用户来说数据库必须提供这样一个“假象”：当前只有这么一个用户连接到了数据库中，这样可以减轻应用层的开发难度。但是，对于数据库系统来说，因为同一时间可能会存在很多用户连接，那么许多并发问题，比如数据竞争（data race），就必须解决。在这样的背景下，数据库管理系统（简称 DBMS）就必须保证并发操作产生的结果是安全的，通过可串行化（serializability）来保证。
tags: ['TiKV', '可串行化', '多版本并发控制', '2PC', '隔离级别']
---


### 并发控制简介
事务隔离在数据库系统中有着非常重要的作用，因为对于用户来说数据库必须提供这样一个“假象”：当前只有这么一个用户连接到了数据库中，这样可以减轻应用层的开发难度。但是，对于数据库系统来说，因为同一时间可能会存在很多用户连接，那么许多并发问题，比如数据竞争（data race），就必须解决。在这样的背景下，数据库管理系统（简称 DBMS）就必须保证并发操作产生的结果是安全的，通过可串行化（serializability）来保证。


虽然 Serilizability 是一个非常棒的概念，但是很难能够有效的实现。一个经典的方法就是使用一种[两段锁（2PL）][1]。通过 2PL，DBMS 可以维护读写锁来保证可能产生冲突的事务按照一个良好的次序（well-defined) 执行，这样就可以保证 Serializability。但是，这种通过锁的方式也有一些缺点：

1. 读锁和写锁会相互阻滞（block）。
2. 大部分事务都是只读（read-only）的，所以从事务序列（transaction-ordering）的角度来看是无害的。如果使用基于锁的隔离机制，而且如果有一段很长的读事务的话，在这段时间内这个对象就无法被改写，后面的事务就会被阻塞直到这个事务完成。这种机制对于并发性能来说影响很大。

**多版本并发控制（Multi-Version Concurrency Control，以下简称 MVCC）** 以一种优雅的方式来解决这个问题。在 MVCC 中，每当想要更改或者删除某个数据对象时，DBMS 不会在原地去删除或这修改这个已有的数据对象本身，而是创建一个该数据对象的新的版本，这样的话同时并发的读取操作仍旧可以读取老版本的数据，而写操作就可以同时进行。这个模式的好处在于，可以让读取操作不再阻塞，事实上根本就不需要锁。这是一种非常诱人的特型，以至于在很多主流的数据库中都采用了 MVCC 的实现，比如说 PostgreSQL，Oracle，Microsoft SQL Server 等。


### TiKV 中的 MVCC
让我们深入到 TiKV 中的 MVCC，了解 MVCC 在 TiKV 中是如何 [实现](https://github.com/pingcap/tikv/tree/master/src/storage) 的。


#### 1. Timestamp Oracle(TSO)
因为`TiKV` 是一个分布式的储存系统，它需要一个全球性的授时服务，下文都称作 TSO（Timestamp Oracle），来分配一个单调递增的时间戳。 这样的功能在 TiKV 中是由 PD 提供的，在 Google 的 [Spanner](http://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf) 中是由多个原子钟和 GPS 来提供的。


#### 2. Storage
从源码结构上来看，想要深入理解 TiKV 中的 MVCC 部分，[src/storage](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mod.rs) 是一个非常好的入手点。 `Storage` 是实际上接受外部命令的结构体。

```rust
pub struct Storage {
    engine: Box<Engine>,
    sendch: SendCh<Msg>,
    handle: Arc<Mutex<StorageHandle>>,
}




impl Storage {
    pub fn start(&mut self, config: &Config) -> Result<()> {
        let mut handle = self.handle.lock().unwrap();
        if handle.handle.is_some() {
            return Err(box_err!("scheduler is already running"));
        }




        let engine = self.engine.clone();
        let builder = thread::Builder::new().name(thd_name!("storage-scheduler"));
        let mut el = handle.event_loop.take().unwrap();
        let sched_concurrency = config.sched_concurrency;
        let sched_worker_pool_size = config.sched_worker_pool_size;
        let sched_too_busy_threshold = config.sched_too_busy_threshold;
        let ch = self.sendch.clone();
        let h = try!(builder.spawn(move || {
            let mut sched = Scheduler::new(engine,
                                           ch,
                                           sched_concurrency,
                                           sched_worker_pool_size,
                                           sched_too_busy_threshold);
            if let Err(e) = el.run(&mut sched) {
                panic!("scheduler run err:{:?}", e);
            }
            info!("scheduler stopped");
        }));
        handle.handle = Some(h);




        Ok(())
    }
}

```

`start` 这个函数很好的解释了一个 storage 是怎么跑起来的。


#### 3. Engine
首先是 [Engine](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/engine/mod.rs#L44)。 `Engine` 是一个描述了在储存系统中接入的的实际上的数据库的接口，[raftkv](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/engine/raftkv.rs#L91) 和 [Enginerocksdb](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/engine/rocksdb.rs#L66) 分别实现了这个接口。


#### 4. StorageHandle
`StorageHanle` 是处理从`sendch` 接受到指令，通过 [mio](https://github.com/carllerche/mio) 来处理 IO。


接下来在`Storage`中实现了`async_get` 和`async_batch_get`等异步函数，这些函数中将对应的指令送到通道中，然后被调度器（scheduler）接收到并异步执行。


Ok，了解完`Storage` 结构体是如何实现的之后，我们终于可以接触到在`Scheduler` [被调用的 MVCC 层](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/txn/scheduler.rs#L763)了。


当 storage 接收到从客户端来的指令后会将其传送到调度器中。然后调度器执行相应的过程或者调用相应的[异步函数](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/txn/scheduler.rs#L643)。在调度器中有两种操作类型，读和写。读操作在 [MvccReader](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mvcc/reader.rs#L20) 中实现，这一部分很容易理解，暂且不表。写操作的部分是MVCC的核心。


#### 5. MVCC
Ok，两段提交（2-Phase Commit，2PC）是在 MVCC 中实现的，整个 TiKV 事务模型的核心。在一段事务中，由两个阶段组成。


##### Prewrite
选择一个 row 作为 primary row， 余下的作为 secondary row。
对primary row [上锁](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mvcc/txn.rs#L80). 在上锁之前，会检查[是否有其他同步的锁已经上到了这个 row 上](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mvcc/txn.rs#L71) 或者是是否经有在 startTS 之后的提交操作。这两种情况都会导致冲突，一旦都冲突发生，就会[回滚（rollback）](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mvcc/txn.rs#L115)。
对于 secondary row 重复以上操作。


##### Commit
[Rollback](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mvcc/txn.rs#L115) 在`Prewrite` 过程中出现冲突的话就会被调用。


##### Garbage Collector
很容易发现，如果没有[垃圾收集器（Gabage Collector）](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/storage/mvcc/txn.rs#L143) 来移除无效的版本的话，数据库中就会存有越来越多的 MVCC 版本。但是我们又不能仅仅移除某个 safe point 之前的所有版本。因为对于某个 key 来说，有可能只存在一个版本，那么这个版本就必须被保存下来。在`TiKV`中，如果在 safe point 前存在 `Put` 或者 `Delete` 记录，那么比这条记录更旧的写入记录都是可以被移除的，不然的话只有`Delete`，`Rollback`和`Lock` 会被删除。



## TiKV-Ctl for MVCC
在开发和 debug 的过程中，我们发现查询 MVCC 的版本信息是一件非常频繁并且重要的操作。因此我们开发了新的工具来查询 MVCC 信息。`TiKV` 将 Key-Value，Locks 和 Writes 分别储存在`CF_DEFAULT`，`CF_LOCK`，`CF_WRITE`中。它们以这样的格式进行编码


|           | default                        | lock                                  | write                           |
|:----------|:-------------------------------|:--------------------------------------|:--------------------------------|
| **key**   | z{encoded_key}{start_ts(desc)} | z{encoded_key}                        | z{encoded_key}{commit_ts(desc)} |
| **value** | {value}                        | {flag}{primary_key}{start_ts(varint)} | {flag}{start_ts(varint)}        |

Details can be found [here](https://github.com/pingcap/tikv/issues/1077).


因为所有的 MVCC 信息在 Rocksdb 中都是储存在 CF Key-Value 中，所以想要查询一个 Key 的版本信息，我们只需要将这些信息以不同的方式编码，随后在对应的 CF 中查询即可。CF Key-Values 的 [表示形式](https://github.com/tikv/tikv/blob/1050931de5d9b47423f997d6fc456bd05bd234a7/src/bin/tikv-ctl.rs#L210)。

[1]: https://en.wikipedia.org/wiki/Two-phase_locking
