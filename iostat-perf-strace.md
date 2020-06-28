---
title: 工欲性能调优，必先利其器（1）
author:
  - 唐刘
date: 2017-05-31
summary: >-
  最近在排查 TiDB
  性能问题的时候，通过工具发现了一些问题，觉得有必要记录一下，让自己继续深刻的去理解相关工具的使用，也同时让同学们对类似问题的时候别再踩坑。
tags:
  - 性能
  - 工具
aliases:
  - /blog-cn/tangliu-tool-1/
---

# 使用 iostat 定位磁盘问题

在一个性能测试集群，我们选择了 AWS c3.4xlarge 机型，主要是为了在一台机器的两块盘上面分别跑 TiKV。在测试一段时间之后，我们发现有一台 TiKV 响应很慢，但是 RocksDB 并没有相关的 Stall 日志，而且慢查询也没有。

于是我登上 AWS 机器，使用 `iostat -d -x -m 5` 命令查看，得到如下输出：

```
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
xvda              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
xvdb              8.00 12898.00  543.00  579.00    31.66    70.15   185.84    51.93   54.39    7.03   98.79   0.60  66.80
xvdc              0.00     0.00  206.00 1190.00    10.58   148.62   233.56   106.67   70.90   13.83   80.78   0.56  78.40
```

上面发现，两个盘 xvdb 和 xvdc 在 wrqm/s 上面差距太大，当然后面一些指标也有明显的差距，这里就不在详细的解释 iostat 的输出。只是需要注意，大家通常将目光注意到 util 上面，但有时候光有 util 是反应不了问题的。

于是我继续用 fio 进行测试，

`fio -ioengine=libaio -bs=4k -direct=1 -thread -rw=write -size=10G -filename=test -name="PingCAP max throughput" -iodepth=4 -runtime=60`

发现两个盘的写入有 2 倍的差距，xvdb 的写入竟然只有不到 70 MB，而 xvdc 有 150 MB，所以自然两个 TiKV 一个快，一个慢了。

对于磁盘来说，通常我们使用的就是 iostat 来进行排查，另外也可以考虑使用 pidstat，iotop 等工具。

# 使用 perf 定位性能问题

RC3 最重要的一个功能就是引入 gRPC，但这个对于 rust 来说难度太大。最开始，我们使用的是 rust-grpc 库，但这个库并没有经过生产环境的验证，我们还是胆大的引入了，只是事后证明，这个冒险的决定还是傻逼了，一些试用的用户跟我们反映 TiKV 时不时 coredump，所以我们立刻决定直接封装 c gRPC。因为现在大部分语言 gRPC 实现都是基于 c gRPC 的，所以我们完全不用担心这个库的稳定性。

在第一个版本的实现中，我们发现，rust 封装的 c gRPC 比 C Plus Plus 的版本差了几倍的性能，于是我用 perf stat 来分别跑 C Plus Plus 和 rust 的benchmark，得到类似如下的输出：

```
Performance counter stats for 'python2.7 tools/run_tests/run_performance_tests.py -r generic_async_streaming_ping_pong -l c++':

     216989.551636 task-clock (msec)         #    2.004 CPUs utilized
         3,659,896 context-switches          #    0.017 M/sec
             5,078 cpu-migrations            #    0.023 K/sec
         4,104,965 page-faults               #    0.019 M/sec
   729,530,805,665 cycles                    #    3.362 GHz
   <not supported> stalled-cycles-frontend
   <not supported> stalled-cycles-backend
   557,766,492,733 instructions              #    0.76  insns per cycle
   121,205,705,283 branches                  #  558.579 M/sec
     3,095,509,087 branch-misses             #    2.55% of all branches

     108.267282719 seconds time elapsed
```

上面是 C Plus Plus 的结果，然后在 rust 测试的时候，我们发现 context-switch 是 C Plus Plus 的 10 倍，也就是我们进行了太多次的线程切换。刚好我们第一个版本的实现是用 rust futures 的 park 和 unpark task 机制，不停的在 gRPC 自己的 Event Loop 线程和逻辑线程之前切换，但 C Plus Plus 则是直接在一个 Event Loop 线程处理的。于是我们立刻改成类似 C Plus Plus 架构，没有了 task 的开销，然后性能一下子跟 C Plus Plus 的不相伯仲了。

当然，perf 能做到的还远远不仅于此，我们通常会使用[火焰图](https://github.com/brendangregg/FlameGraph)工具，关于火焰图，网上已经有太多的介绍，我们也通过它来发现了很多性能问题，这个后面可以专门来说一下。

# 使用 strace 动态追踪

因为我们有一个记录线程 CPU 的统计，通常在 Grafana 展示的时候都是按照线程名字来 group 的，并没有按照线程 ID。但我们也可以强制发送 SIGUSR1 信号给 TiKV 在 log 里面 dump 相关的统计信息。在测试 TiKV 的时候，我发现 pd worker 这个 thread 出现了很多不同线程 ID 的 label，也就是说，这个线程在不停的创建和删除。

要动态追踪到这个情况，使用 `strace -f` 是一个不错的方式，我同时观察 TiKV 自己的输出 log，发现当 TiKV 在处理分裂逻辑，给 PD worker 发送 message 的时候，就有一个新的线程创建出来。然后在查找对应的代码，发现我们每次在发消息的时候都创建了一个 tokio-timer，而这个每次都会新创建一个线程。

有时候，也可以使用 `strace -c` 来动态的追踪一段时间的系统调用。在第一版本的 rust gRPC 中，我们为了解决 future task 导致的频繁线程切换，使用 gRPC 自己的 alarm 来唤醒 Event Loop，但发现这种实现会产生大量的信号调用，因为 gRPC 的 alarm 会发送一个实时信号用来唤醒 epoll，后面通过火焰图也发现了 Event Loop 很多 CPU 消耗在 alarm 这边，所以也在开始改进。

这里需要注意，strace 对性能影响比较大，但对于内部性能测试影响还不大，不到万不得已，不建议长时间用于生产环境。

# 小结

上面仅仅是三个最近用工具发现的问题，当然还远远不止于此，后续也会慢慢补上。其实对于性能调优来说，工具只是一个辅助工具，最重要的是要有一颗对问题敏锐的心，不然即使工具发现了问题，因为不敏锐直接就忽略了。我之前就是不敏锐栽过太多的坑，所以现在为了刻意提升自己这块的能力，直接给自己下了死规定，就是怀疑一切能能怀疑的东西，认为所有东西都是有问题的。即使真的是正常的，也需要找到充足的理由去验证。
