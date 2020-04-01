---
title: TiKV 源码解析系列文章（六）raft-rs 日志复制过程分析
author: ['屈鹏']
date: 2019-04-24
summary: 本文将对数据冗余复制的过程进行详细展开，特别是关于 snapshot 及流量控制的机制，帮助读者更深刻地理解 Raft 的原理。
tags: ['TiKV 源码解析','社区']
---

在 [《TiKV 源码解析系列文章（二）raft-rs proposal 示例情景分析》 ](https://pingcap.com/blog-cn/tikv-source-code-reading-2/) 中，我们主要介绍了 raft-rs 的基本 API 使用，其中，与应用程序进行交互的主要 API 是：

1. RawNode::propose 发起一次新的提交，尝试在 Raft 日志中追加一个新项；

2. RawNode::ready_since 从 Raft 节点中获取最近的更新，包括新近追加的日志、新近确认的日志，以及需要给其他节点发送的消息等；

3. 在将一个 Ready 中的所有更新处理完毕之后，使用 RawNode::advance 在这个 Raft 节点中将这个 Ready 标记为完成状态。

熟悉了以上 3 个 API，用户就可以写出基本的基于 Raft 的分布式应用的框架了，而 Raft 协议中将写入同步到多个副本中的任务，则由 raft-rs 库本身的内部实现来完成，无须应用程序进行额外干预。本文将对数据冗余复制的过程进行详细展开，特别是关于 snapshot 及流量控制的机制，帮助读者更深刻地理解 Raft 的原理。

## 一般 MsgAppend 及 MsgAppendResponse 的处理

在 Raft leader 上，应用程序通过 RawNode::propose 发起的写入会被处理成一条 MsgPropose 类型的消息，然后调用 Raft::append_entry 和 Raft::bcast_append 将消息中的数据追加到 Raft 日志中并广播到其他副本上。整体流程如伪代码所示：

```rust
fn Raft::step_leader(&mut self, mut m: Message) -> Result<()> {
    if m.get_msg_type() == MessageType::MsgPropose {
        // Propose with an empty entry list is not allowed.
        assert!(!m.get_entries().is_empty());
        self.append_entry(&mut m.mut_entries());
        self.bcast_append();
    }
}
```

这段代码中 `append_entry` 的参数是一个可变引用，这是因为在 `append_entry` 函数中会为每一个 Entry 赋予正确的  term 和 index。term 由选举产生，在一个 Raft 系统中，每选举出一个新的 Leader，便会产生一个更高的 term。而 index 则是 Entry 在 Raft 日志中的下标。Entry 需要带上 term 和 index 的原因是，在其他副本上的 Raft 日志是可能跟 Leader 不同的，例如一个旧 Leader 在相同的位置（即 Raft 日志中具有相同 index 的地方）广播了一条过期的 Entry，那么当其他副本收到了重叠的、但是具有更高 term 的消息时，便可以用它们替换旧的消息，以便达成与最新的 Leader 一致的状态。

在 Leader 将新的写入追加到自己的 Raft log 中之后，便可以调用 `bcast_append` 将它们广播到其他副本了。注意这个函数并没有任何参数，那么 Leader 如何知道应该给每一个副本从哪一个位置开始广播呢？原来在 Leader 上对每一个副本，都关联维护了一个 Progress，该结构体定义如下：

```rust
pub struct Progress {
    pub matched: u64,
    // 该副本期望接收的下一个 Entry 的 index
    pub next_idx: u64,
    // 未 commit 的消息的滑动窗口
    pub ins: Inflights,
    // ProgressState::Probe：Leader 每个心跳间隔中最多发送一条 MsgAppend
    // ProgressState::Replicate：Leader 在每个心跳间隔中可以发送多个 MsgAppend
    // ProgressState::Snapshot：Leader 无法再继续发送 MsgAppend 给这个副本
    pub state: ProgressState,
    // 是否暂停给这个副本发送 MsgAppend 了
    pub paused: bool,
    // 一些其他字段……
}
```


如代码注释中所说的那样，Leader 在给副本广播新的日志时，会从对应的副本的 `next_idx` 开始。这就蕴含了两个问题：

1.  在刚开始启动的时候，所有副本的 `next_idx` 应该如何设置？
2.  在接收并处理完成 Leader 广播的新写入后，其他副本应该如何向 Leader 更新 `next_idx`？

第一个问题的答案在 `Raft::reset` 函数中。这个函数会在 Raft 完成选举之后选出的 Leader 上调用，会将 Leader 的所有其他副本的 `next_idx` 设置为跟 Leader 相同的值。之后，Leader 就可以会按照 Raft 论文里的规定，广播一条包含了自己的 term 的空 Entry 了。

第二个问题的答案在 `Raft::handle_append_response` 函数中。我们继续考察上面的情景，Leader 的其他副本在收到 Leader 广播的最新的日志之后，可能会采取两种动作：

```rust
fn Raft::handle_append_entries(&mut self, m: &Message) {
    let mut to_send = Message::new_message_append_response();
    match self.raft_log.maybe_append(...) {
        // 追加日志成功，将最新的 last index 上报给 Leader
        Some(last_index) => to_send.set_index(last_index),
        // 追加日志失败，设置 reject 标志，并告诉 Leader 自己的 last index
        None => {
            to_send.set_reject(true);
            to_send.set_reject_hint(self.raft_log.last_index());
        }
    }
}
self.send(to_send);
```


其他副本调用 `maybe_append` 失败的原因可能是比 Leader 的日志更少，但是 Leader 在刚选举出来的时候将所有副本的 `next_idx` 设置为与自己相同的值了。这个时候这些副本就会在 MsgAppendResponse 中设置拒绝的标志。在 Leader 接收到这样的反馈之后，就可以将对应副本的 `next_idx` 设置为正确的值了。这个逻辑在 `Raft::handle_append_response` 中：

```rust
fn Raft::handle_append_response(&mut self, m: &Message, …) {
    if m.get_reject() {
        let pr: &mut Progress = self.get_progress(m.get_from());
        // 将副本对应的 `next_idx` 回退到一个合适的值
        pr.maybe_decr_to(m.get_index(), m.get_reject_hint());
    } else {
        // 将副本对应的 `next_idx` 设置为 `m.get_index() + 1`
        pr.maybe_update(m.get_index());
    }
}
```


以上伪代码中我们省略了一些丢弃乱序消息的代码，避免过多的细节造成干扰。

## pipeline 优化和流量控制机制

上一节我们重点观察了 MsgAppend 及 MsgAppendResponse 消息的处理流程，原理是非常简单、清晰的。然而，这个未经任何优化的实现能够工作的前提是在 Leader 收到某个副本的 MsgAppendResponse 之前，不再给它发送任何 MsgAppend。由于等待响应的时间取决于网络的 TTL，这在实际应用中是非常低效的，因此我们需要引入 pipeline 优化，以及配套的流量控制机制来避免“优化”带来的网络壅塞。

Pipeline 在 `Raft::prepare_send_entries` 函数中被引入。这个函数在 `Raft::send_append` 中被调用，内部会直接修改对目标副本的 `next_idex` 值，这样，后续的 MsgAppend 便可以在此基础上继续发送了。而一旦之前的 MsgAppend 被该目标副本拒绝掉了，也可以通过上一节中介绍的 `maybe_decr_to` 机制将 `next_idx` 重置为正确的值。我们来看一下这段代码：

```rust
// 这个函数在 `Raft::prepare_send_entries` 中被调用
fn Progress::update_state(&mut self, last: u64) {
    match self.state {
        ProgressState::Replicate => {
            self.next_idx = last + 1;
            self.ins.add(last);
        },
        ProgressState::Probe => self.pause(),
       _ => unreachable!(),
    }
}
```

Progress 有 3 种不同的状态，如这个结构体的定义的代码片段所示。其中 Probe 状态和 Snapshot 状态会在下一节详细介绍，现在只需要关注 Replicate 状态。我们已经知道 Pipeline 机制是由更新 `next_idx` 的那一行引入的了，那么下面更新 `ins` 的一行的作用是什么呢？

从 Progress 的定义的代码片段中我们知道，`ins` 字段的类型是 Inflights，可以想象成一个类似 TCP 的滑动窗口：所有 Leader 发出了，但是尚未被目标副本响应的消息，都被框在该副本在 Leader 上对应的 Progress 的 `ins` 中。这样，由于滑动窗口的大小是有限的，Raft 系统中任意时刻的消息数量也会是有限的，这就实现了流量控制的机制。更具体地，Leader 在给某一副本发送 MsgAppend 时，会检查其对应的滑动窗口，这个逻辑在 `Raft::send_append` 函数中；在收到该副本的 MsgAppendResponse 之后，会适时调用 Inflights 的 `free_to` 函数，使窗口向前滑动，这个逻辑在 `Raft::handle_append_response` 中。

## ProgressState 相关优化

我们已经在 Progress 结构体的定义以及上面一些代码片段中见过了 ProgressState 这个枚举类型。在 3 种可能的状态中，Replicate 状态是最容易理解的，Leader 可以给对应的副本发送多个 MsgAppend 消息（不超过滑动窗口的限制），并适时地将窗口向前滑动。然而，我们注意到，在 Leader 刚选举出来时，Leader 上面的所有其他副本的状态却被设置成了 Probe。这是为什么呢？

从 Progress 结构体的字段注释中，我们知道当某个副本处于 Probe 状态时，Leader 只能给它发送 1 条 MsgAppend 消息。这是因为，在这个状态下的 Progress 的 `next_idx` 是 Leader 猜出来的，而不是由这个副本明确的上报信息推算出来的。它有很大的概率是错误的，亦即 Leader 很可能会回退到某个地方重新发送；甚至有可能这个副本是不活跃的，那么 Leader 发送的整个滑动窗口的消息都可能浪费掉。因此，我们引入 Probe 状态，当 Leader 给处于这一状态的副本发送了 MsgAppend 时，这个 Progress 会被暂停掉（源码片段见上一节），这样在下一次尝试给这个副本发送 MsgAppend 时，会在 `Raft::send_append` 中跳过。而当 Leader 收到了这个副本上报的正确的 last index 之后，Leader 便知道下一次应该从什么位置给这个副本发送日志了，这一过程在 `Progress::maybe_update` 函数中：

```rust
fn Progress::maybe_update(&mut self, n: u64) {
    if self.matched < n {
        self.matched = n;
        self.resume(); // 取消暂停的状态
    }
    if self.next_idx < n + 1 {
        self.next = n + 1;
    }
}
```

ProgressState::Snapshot 状态与 Progress 中的 pause 标志十分相似，一个副本对应的 Progress 一旦处于这个状态，Leader 便不会再给这个副本发送任何 MsgAppend 了。但是仍有细微的差别：事实上在 Leader 收到 MsgHeartbeatResponse 时，也会调用 `Progress::resume` 来将取消对该副本的暂停，然而对于 ProgressState::Snapshot 状态的 Progress 则没有这个逻辑。这个状态会在 Leader 成功发送完成 Snapshot，或者收到了对应的副本的最新的 MsgAppendResponse 之后被改变，详细的逻辑请参考源代码，这里就不作赘述了。

我们把篇幅留给在 Follower 上收到 Snapshot 之后的处理逻辑，主要是 `Raft::restore_raft` 和 `RaftLog::restore` 两个函数。前者中主要包含了对 Progress 的处理，因为 Snapshot 包含了 Leader 上最新的信息，而 Leader 上的 Configuration 是可能跟 Follower 不同的。后者的主要逻辑伪代码如下所示：

```rust
fn RaftLog::restore(&mut self, snapshot: Snapshot) {
    self.committed = snapshot.get_metadata().get_index();
    self.unstable.restore(snapshot);
}
```

可以看到，内部仅更新了 committed，并没有更新 applied。这是因为 raft-rs 仅关心 Raft 日志的部分，至于如何把日志中的内容更新到真正的状态机中，是应用程序的任务。应用程序需要从上一篇文章中介绍的 Ready 接口中把 Snapshot 拿到，然后自行将其应用到状态机中，最后再通过 `RawNode::advance` 接口将 applied 更新到正确的值。

## 总结

Raft 日志复制及相关的流量控制、Snapshot 流程就介绍到这里，代码仓库仍然在 [https://github.com/pingcap/raft-rs](https://github.com/pingcap/raft-rs)，source-code 分支。下一期 raft-rs 源码解析我们会继续为大家带来 configuration change 相关的内容，敬请期待！
