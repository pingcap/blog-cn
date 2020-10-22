---
title: 我们为什么要禁用 THP
author: ['张文博']
date: 2020-10-22
summary: 本文将和大家分享 THP 引起性能抖动的原因、典型的现象，分析方法等，在文章的最后给出使用 THP 时的配置建议及关闭方法。
tags: ['THP','TiDB']
---

## 前言

我们之前在生产环境上遇到过很多起由操作系统的某些特征引起的性能抖动案例，其中 THP 作案次数较多，因此本文将和大家分享 THP 引起性能抖动的原因、典型的现象，分析方法等，在文章的最后给出使用 THP 时的配置建议及关闭方法。

## THP（Transparent Huge Page） 简介

世界并不是非黑即白的，THP 也是内核的一个重要特征，且持续在演进，其目的是通过将页表项映射更大的内存，来减少 Page Fault，从而提升 TLB （Translation Lookaside Buffer，由存储器管理单元用于改进虚拟地址到物理地址的转译速度）的命中率。结合存储器层次结构设计原理可知，当程序的访存局部性较好时，THP 将带来性能提升，反之 THP 的优势不仅丧失，还有可能化身为恶魔，引起系统的不稳定。遗憾的是数据库的负载访问特征通常是离散的。

## Linux 内存管理回顾

在陈述 THP 引起的负面现象前，先来和大家一起回忆下，Linux 操作系统是如何管理物理内存的。

对于不同的体系结构，内核对应不同的内存布局图。其中用户空间通过多级页表进行映射来节约映射管理所需的空间，而内核空间为了简单高效采用线性映射。在内核启动时，物理页面将加入到伙伴系统 （Buddy System）中，用户申请内存时分配，释放时回收。为了照顾慢速设备及兼顾多种 workload，Linux 将页面类型分为匿名页（Anon Page）和文件页 （Page Cache），及 swapness，使用 Page Cache 缓存文件 （慢速设备），通过 swap cache 和 swapness 交由用户根据负载特征决定内存不足时回收二者的比例。

为了尽可能快的响应用户的内存申请需求并保证系统在内存资源紧张时运行，Linux 定义了三条水位线 （high，low，min），当剩余物理内存低于 low 高于 min 水位线时，在用户申请内存时通过 kswapd 内核线程异步回收内存，直到水位线恢复到 high 以上，若异步回收的速度跟不上线程内存申请的速度时，将触发同步的直接内存回收，也就是所有申请内存的线程都同步的参与内存回收，一起将水位线抬上去后再获得内存。这时，若需要回收的页面是干净的，则同步引起的阻塞时间就比较短，反之则很大（比如几十、几百ms 甚至 s 级，取决于后端设备速度）。

除水位线外，当申请大的连续内存时，若剩余物理内存充足，但碎片化比较严重时，内核在做内存规整的时候，也有可能触发直接内存回收（取决于碎片化指数，后面会介绍）。因此内存直接回收和内存规整是进程申请内存路径上的可能遇到的主要延迟。而在访存局部性差的负载下，THP 将成为触发这两个事件的幕后黑手。

## 最典型特征 —— Sys CPU 使用率飙升

我们在多个用户现场发现当分配 THP 引发性能波动时，其最典型的特征就是 Sys CPU 使用率飙升，这种特征的分析比较简单，通过 perf 抓取 on-cpu火焰图，我们就可以看到我们服务所有处于 R 状态的线程都在做内存规整，且缺页异常处理函数为 do_huge_pmd_anonymous_page，说明当前系统没有连续 2M 的物理内存，因此触发了直接内存规整，直接内存规整的逻辑是很耗时的，是导致 sys 利用率升高的原因。
 
## 间接特征—— Sys load 飙升

真实的系统往往是复杂的，当分配 THP 或分配其它高阶内存时，系统并不会做直接内存规整，留下上述那么典型的犯罪特征，而是混合其他行为，比如直接内存回收。直接内存回收的参与让事情变的稍微有些复杂和令人疑惑，比如我们最初从客户现场看到 normal zone 的剩余物理内存高于 high 水位线，可系统为啥不停的在做直接内存回收呢？我们深入到慢速内存分配的处理逻辑中可知，慢速内存分配路径主要有几个步骤：

1. 异步内存规整；

2. 直接内存回收；

3. 直接内存规整；

4. oom 回收。

每个步骤处理完成后，都会尝试分配内存，如果可分配了，则直接返回页面，略过后面的部分。其中内核为伙伴系统的每个 order 提供了碎片指数来表示内存分配失败是由于内存不足还是碎片化引起的。和其关联的是 /proc/sys/vm/extfrag_threshold，当接近 1000 时，表示分配失败主要和碎片化相关，此时内核会倾向于做内存规整，当接近 0 时，表示分配失败和内存不足关联更大，则内核会倾向于做内存回收。因此产生了在高于 high 水位线的时候，频繁进行直接内存回收的现象 。而由于 THP 的开启和使用占据了高阶内存，因此加速了内存碎片化引起的性能抖动问题。

对此特征，判定方法如下：

1.  运行 sar -B 观察 pgscand/s，其含义为每秒发生的直接内存回收次数，当在一段时间内持续大于 0 时，则应继续执行后续步骤进行排查；

2. 运行 `cat /sys/lernel/debug/extfrag/extfrag_index` 观察内存碎片指数，重点关注 order >= 3 的碎片指数，当接近 1.000 时，表示碎片化严重，当接近 0 时表示内存不足；

3. 运行 `cat /proc/buddyinfo, cat /proc/pagetypeinfo` 查看内存碎片情况， 指标含义参考 （https://man7.org/linux/man-pages/man5/proc.5.html），同样关注 order >= 3 的剩余页面数量，pagetypeinfo 相比 buddyinfo 展示的信息更详细一些，根据迁移类型 （伙伴系统通过迁移类型实现反碎片化）进行分组，需要注意的是，当迁移类型为 Unmovable 的页面都聚集在 order < 3 时，说明内核 slab 碎片化严重，我们需要结合其他工具来排查具体原因，在本文就不做过多介绍了；

4. 对于 CentOS 7.6 等支持 BPF 的 kernel 也可以运行我们研发的 [drsnoop](https://github.com/iovisor/bcc/blob/master/tools/drsnoop_example.txt)，[compactsnoop](https://github.com/iovisor/bcc/blob/master/tools/compactsnoop_example.txt) 工具对延迟进行定量分析，使用方法和解读方式请参考对应文档；

5. (Opt) 使用 ftrace 抓取 mm_page_alloc_extfrag  事件，观察因内存碎片从备用迁移类型“盗取”页面的信息。

## 非典型特征—— 异常的 RES 使用率

我们在 AARCH64 服务器上，遇到过服务刚启动就占用几十个 G 物理内存的场景，通过观察 /proc/pid/smaps 文件可以看到内存大部分用于 THP， 且 AARCH64 的 CentOS 7 内核编译时选用的 PAGE SIZE 为 64K，因此相比 X86_64 平台的内存用量差出很多倍。在定位的过程中我们也顺便修复了 jemalloc 未完全关闭 THP 的 bug: [fix opt.thp:never still use THP with base_map](https://github.com/jemalloc/jemalloc/pull/1704)。

## 结语

对于未对访存局部性进行优化的程序或负载本身就是离散的访存程序而言，将 THP 以及 THP defrag 设置为始终开启，对长时间运行的服务而言有害无益，且内核从 4.6 版本内核版本起才对 THP 的 defrag 提供了 defer，defer + madvise 等优化。因此对于我们常用的 CentOS 7 3.10 版本的内核来说，若程序需要使用 THP，则建议将 THP 的开关设置为 madvise，在程序中通过 madvise 系统调用来分配 THP， 否则设置成 never 禁用掉是最佳选择：

查看当前的 THP 模式：

```
cat /sys/kernel/mm/transparent_hugepage/enabled
```

若值是 always 时，执行：

```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

完成关闭操作。

需要注意的是为防止服务器重启失效，应将这两个命令写入到 .sevice 文件中，交给 systemd 进行管理。