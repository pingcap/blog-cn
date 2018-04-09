---
title: 工欲性能调优，必先利其器（2）- 火焰图
author: ['唐刘']
date: 2017-06-26
summary: 本篇文章将介绍一下，我们在 TiKV 性能调优上面用的最多的工具 - 火焰图。
tags: ['性能', '工具']
aliases:
  - /blog-cn/tangliu-tool-2/
---

在[前一篇](./iostat-perf-strace.md)文章，我们简单提到了 perf，实际 perf 能做的事情远远不止这么少，这里就要好好介绍一下，我们在 TiKV 性能调优上面用的最多的工具 - 火焰图。

火焰图，也就是 [FlameGraph](https://github.com/brendangregg/FlameGraph)，是超级大牛 Brendan Gregg 捣鼓出来的东西，主要就是将 profile 工具生成的数据进行可视化处理，方便开发人员查看。我第一次知道火焰图，应该是来自 OpenResty 的章亦春介绍，大家可以详细去看看这篇文章[动态追踪技术漫谈](https://openresty.org/posts/dynamic-tracing/)。

之前，我的所有工作在很长一段时间几乎都是基于 Go 的，而 Go 原生提供了很多相关的 profile 工具，以及可视化方法，所以我没怎么用过火焰图。但开始用 Rust 开发 TiKV 之后，我就立刻傻眼了，Rust 可没有官方的工具来做这些事情，怎么搞？自然，我们就开始使用火焰图了。

使用火焰图非常的简单，我们仅仅需要将代码 clone 下来就可以了，我通常喜欢将相关脚本扔到 `/opt/FlameGraph` 下面，后面也会用这个目录举例说明。

一个简单安装的例子：

```
  wget https://github.com/brendangregg/FlameGraph/archive/master.zip
  unzip master.zip
  sudo mv FlameGraph-master/ /opt/FlameGraph
```

## CPU

对于 TiKV 来说，性能问题最开始关注的就是 CPU，毕竟这个是一个非常直观的东西。

当我们发现 TiKV CPU 压力很大的时候，通常会对 TiKV 进行 perf，如下：

```
  perf record -F 99 -p tikv_pid -g -- sleep 60
  perf script > out.perf
```

上面，我们对一个 TiKV 使用 99 HZ 的频繁采样 60 s，然后生成对应的采样文件。然后我们生成火焰图：

```
/opt/FlameGraph/stackcollapse-perf.pl out.perf > out.folded
/opt/FlameGraph/flamegraph.pl out.folded > cpu.svg
```

![][1]

上面就是生成的一个 TiKV 火焰图，我们会发现 gRPC 线程主要开销在 c gRPC core 上面，而这个也是现在 c gRPC core 大家普遍反映的一个问题，就是太费 CPU，但我相信凭借 Google gRPC team 的实力，这问题应该能够搞定。

另外，在 gRPC 线程上面，我们可以发现，protobuf 的编解码也占用了很多 CPU，这个也是现阶段 rust protobuf 库的一个问题，性能比较差，但幸好后面的办法有一个优化，我们准备马上采用。

另外，还需要注意，raftstore 线程主要的开销在于 RocksDB 的 Get 和 Write，对于 TiKV 来说，如果 raftstore 线程出现了瓶颈，那么整个 Raft 流程都会被拖慢，所以自然这个线程就是我们的重点优化对象。

可以看到，Get 的开销其实就是我们拿到 Raft 的 committed entries，然后扔给 apply Raft log 线程去异步 apply，所以自然这一步 Get 自然能扔到 apply worker 去处理。另外，对于 Write，鉴于 Raft log 的格式，我们可以非常方便的使用 RocksDB 一个 `insert_with_hint` 特性来优化，或者将 Write 也放到另一个线程去 async 处理。

可以看到，我们通过火焰图，能够非常方便的发现 CPU 大部分时间开销都消耗在哪里，也就知道如何优化了。

这里在说一下，大家通常喜欢将目光定在 CPU 消耗大头的地方，但有时候一些小的不起眼的地方，也需要引起大家的注意。这里并不是这些小地方会占用多少 CPU，而是要问为啥会在火焰图里面出现，因为按照正常逻辑是不可能的。我们通过观察 CPU 火焰图这些不起眼的小地方，至少发现了几处代码 bug。

## Memory

通常大家用的最多的是 CPU 火焰图，毕竟这个最直观，但火焰图可不仅仅只有 CPU 的。我们还需要关注除了 CPU 之外的其他指标。有一段时间，我对 TiKV 的内存持续上涨问题一直很头疼，虽然 TiKV 有 OOM，但总没有很好的办法来定位到底是哪里出了问题。于是也就研究了一下 memory 火焰图。

要 profile TiKV 的 memory 火焰图，其实我们就需要监控 TiKV 的 malloc 分配，只要有 malloc，就表明这时候 TiKV 在进行内存分配。因为 TiKV 是自己内部使用了 jemalloc，并没有用系统的 malloc，所以我们不能直接用 perf 来探查系统的 malloc 函数。幸运的是，perf 能支持动态添加探针，我们将 TiKV 的 malloc 加入：

```
perf probe -x /deploy/bin/tikv-server -a malloc
```

然后采样生成火焰图:

```
perf record -e probe_tikv:malloc -F 99 -p tikv_pid -g -- sleep 10
perf script > out.perf
/opt/FlameGraph/stackcollapse-perf.pl out.perf > out.folded
/opt/FlameGraph/flamegraph.pl  --colors=mem out.folded > mem.svg
```

![][2]

上面是生成的一个 malloc 火焰图，我们可以看到，大部分的内存开销仍然是在 RocksDB 上面。

通过 malloc 火焰图，我们曾发现过 RocksDB 的 ReadOption 在非常频繁的调用分配，后面准备考虑直接在 stack 上面分配，不过这个其实对性能到没啥太大影响 :sweat: 。

除了 malloc，我们也可以 probe minor page fault 和 major page fault，因为用 pidstat 观察发现 TiKV 很少有 major page fault，所以我们只 probe 了 minor，如下：

```
perf record -e minor-faults -F 99 -p $1 -g -- sleep 10
perf script > out.perf
/opt/FlameGraph/stackcollapse-perf.pl out.perf > out.folded
/opt/FlameGraph/flamegraph.pl  --colors=mem out.folded > minflt.svg
```

## Off CPU

有时候，我们还会面临一个问题。系统的性能上不去，但 CPU 也很闲，这种的很大可能都是在等 IO ，或者 lock 这些的了，所以我们需要看到底 CPU 等在什么地方。

对于 perf 来说，我们可以使用如下方式采样 off CPU。

```
perf record -e sched:sched_stat_sleep -e sched:sched_switch \
    -e sched:sched_process_exit -p tikv_pid -g -o perf.data.raw sleep 10
perf inject -v -s -i perf.data.raw -o perf.data
```

但不幸的是，上面的代码在 Ubuntu 或者 CentOS 上面通常都会失败，主要是现在最新的系统为了性能考虑，并没有支持 sched statistics。 对于 Ubuntu，貌似只能重新编译内核，而对于 CentOS，只需要安装 kernel debuginfo，然后在打开 sched statistics 就可以了，如下:

```
dnf install kernel-debug kernel-debug-devel kernel-debug-debuginfo
echo 1 | sudo tee /proc/sys/kernel/sched_schedstats
```

然后生成 off cpu 火焰图:

```
perf script -F comm,pid,tid,cpu,time,period,event,ip,sym,dso,trace | awk '
    NF > 4 { exec = $1; period_ms = int($5 / 1000000) }
    NF > 1 && NF <= 4 && period_ms > 0 { print $2 }
    NF < 2 && period_ms > 0 { printf "%s\n%d\n\n", exec, period_ms }' | \
    /opt/FlameGraph/stackcollapse.pl | \
    /opt/FlameGraph/flamegraph.pl --countname=ms --title="Off-CPU Time Flame Graph" --colors=io > offcpu.svg
```

![][3]

上面就是 TiKV 一次 off CPU 的火焰图，可以发现只要是 server event loop 和 time monitor 两个线程 off CPU 比较长，server event loop 是等待外部的网络请求，因为我在 perf 的时候并没有进行压力测试，所以 wait 是正常的。而 time monitor 则是 sleep 一段时间，然后检查时间是不是出现了 jump back，因为有长时间的 sleep，所以也是正常的。

上面我说到，对于 Ubuntu 用户，貌似只能重新编译内核，打开 sched statistics，如果不想折腾，我们可以通过 systemtap 来搞定。systemtap 是另外一种 profile 工具，其实应该算另一门语言了。

我们可以直接使用 OpenResty 的 systemtap 工具，来生成 off CPU 火焰图，如下：

```
wget https://raw.githubusercontent.com/openresty/openresty-systemtap-toolkit/master/sample-bt-off-cpu
chmod +x sample-bt-off-cpu

./sample-bt-off-cpu -t 10 -p 13491 -u > out.stap
/opt/FlameGraph/stackcollapse-stap.pl out.stap > out.folded
/opt/FlameGraph/flamegraph.pl --colors=io out.folded > offcpu.svg
```

可以看到，使用 systemptap 的方式跟 perf 没啥不一样，但 systemtap 更加复杂，毕竟它可以算是一门语言。而 FlameGraph 里面也自带了 systemaptap 相关的火焰图生成工具。

## Diff 火焰图

除了通常的几种火焰图，我们其实还可以将两个火焰图进行 diff，生成一个 diff 火焰图，如下：

```
/opt/difffolded.pl out.folded1 out.folded2 | ./flamegraph.pl > diff2.svg
```

![][4]

但现在我仅仅只会生成，还没有详细对其研究过，这里就不做过多说明了。

## 总结

上面简单介绍了我们在 TiKV 里面如何使用火焰图来排查问题，现阶段主要还是通过 CPU 火焰图发现了不少问题，但我相信对于其他火焰图的使用研究，后续还是会很有帮助的。


  [1]: http://static.zybuluo.com/zyytop/yduq8ncg6ja4s4310wg4xk2p/cpu.jpg
  [4]: http://static.zybuluo.com/zyytop/twmscpi5uixcm5ny0n29od6h/diff2.png
  [2]: http://static.zybuluo.com/zyytop/5p110cw96xnpfh0cuwwkt7wf/mem.png
  [3]: http://static.zybuluo.com/zyytop/mg0s8dpac6tm8j59p2meyooo/offcpu.png
