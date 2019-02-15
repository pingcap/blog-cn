---
title: TiDB 源码阅读系列文章（十四）统计信息（下）
author: ['谢海滨']
date: 2018-07-18
summary: 本篇文章将介绍直方图和 Count-Min(CM) Sketch 的数据结构，然后介绍 TiDB 是如何实现统计信息的查询、收集以及更新的。
tags: ['源码阅读','TiDB','社区']
---


在 [统计信息（上）](https://pingcap.com/blog-cn/tidb-source-code-reading-12/) 中，我们介绍了统计信息基本概念、TiDB 的统计信息收集/更新机制以及如何用统计信息来估计算子代价，本篇将会结合原理介绍 TiDB 的源码实现。

文内会先介绍直方图和 Count-Min(CM) Sketch 的数据结构，然后介绍 TiDB 是如何实现统计信息的查询、收集以及更新的。

## 数据结构定义 

直方图的定义可以在 [histograms.go](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L40) 中找到，值得注意的是，对于桶的上下界，我们使用了在 [《TiDB 源码阅读系列文章（十）Chunk 和执行框架简介》](https://pingcap.com/blog-cn/tidb-source-code-reading-10/) 中介绍到 Chunk 来存储，相比于用 Datum 的方式，可以减少内存分配开销。

CM Sketch 的定义可以在 [cmsketch.go](https://github.com/lamxTyler/tidb/blob/source-code/statistics/cmsketch.go#L31) 中找到，比较简单，包含了 CM Sketch 的核心——二维数组 `table`，并存储了其深度与宽度，以及总共插入的值的数量，当然这些都可以直接从 `table` 中得到。

除此之外，对列和索引的统计信息，分别使用了 [Column](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L699) 和 [Index](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L773) 来记录，主要包含了直方图，CM Sketch 等。 

## 统计信息创建

在执行 analyze 语句时，TiDB 会收集直方图和 CM Sketch 的信息。在执行 analyze 命令时，会先将需要 analyze 的列和索引在 [builder.go](https://github.com/lamxTyler/tidb/blob/source-code/plan/planbuilder.go#L609) 中切分成不同的任务，然后在 [analyze.go](https://github.com/lamxTyler/tidb/blob/source-code/executor/analyze.go#L114) 中将任务下推至 TiKV 上执行。由于在 TiDB 中也包含了 TiKV 部分的实现，因此在这里还是会以 TiDB 的代码来介绍。在这个部分中，我们会着重介绍直方图的创建。

### 列直方图的创建

在统计信息（上）中提到，在建立列直方图的时候，会先进行抽样，然后再建立直方图。

在 [collect](https://github.com/lamxTyler/tidb/blob/source-code/statistics/sample.go#L113) 函数中，我们实现了蓄水池抽样算法，用来生成均匀抽样集合。由于其原理和代码都比较简单，在这里不再介绍。

采样完成后，在 [BuildColumn](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L97) 中，我们实现了列直方图的创建。首先将样本排序，确定每个桶的高度，然后顺序遍历每个值 V：

* 如果 V 等于上一个值，那么把 V 放在与上一个值同一个桶里，无论桶是不是已经满，这样可以保证每个值只存在于一个桶中。

* 如果不等于上一个值，那么判断当前桶是否已经满，就直接放入当前桶，并用 [updateLastBucket](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L146) 更改桶的上界和深度。

* 否则的话，用 [AppendBucket](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L151) 放入一个新的桶。

### 索引直方图的创建

在建立索引列直方图的时候，我们使用了 [SortedBuilder](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L24) 来维护建立直方图的中间状态。由于不能事先知道有多少行的数据，也就不能确定每一个桶的深度，不过由于索引列的数据是已经有序的，因次我们在 [NewSortedBuilder](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L38) 中将每个桶的初始深度设为 1。对于每一个数据，[Iterate](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L50) 会使用建立列直方图时类似的方法插入数据。如果在某一时刻，所需桶的个数超过了当前桶深度，那么用 [mergeBucket](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L74) 将之前的每两个桶合并为 1 个，并将桶深扩大一倍，然后继续插入。

在收集了每一个 Region 上分别建立的直方图后，还需要用 [MergeHistogram](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L609) 把每个 Region 上的直方图进行合并。在这个函数中：

* 为了保证每个值只在一个桶中，我们处理了一下交界处桶的问题，即如果交界处两个桶的上界和下界 [相等](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L623)，那么需要先合并这两个桶；

* 在真正合并前，我们分别将两个直方图的平均桶深 [调整](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L642) 至大致相等；

* 如果直方图合并之后桶的个数超过了限制，那么把两两相邻的桶 [合二为一](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L653)。

## 统计信息维护

在 [统计信息（上）](https://pingcap.com/blog-cn/tidb-source-code-reading-12/) 中，我们介绍了 TiDB 是如何更新直方图和 CM Sketch 的。对于 CM Sketch 其更新比较简单，在这里不再介绍。这个部分主要介绍一下 TiDB 是如何收集反馈信息和维护直方图的。

### 反馈信息的收集

统计信息（上）中提到，为了不去假设所有桶贡献的误差都是均匀的，需要收集每一个桶的反馈信息，因此需要先把查询的范围按照直方图桶的边界切分成不相交的部分。

在 [SplitRange](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L511) 中，我们按照直方图去切分查询的范围。由于目前直方图中的一个桶会包含上下界，为了方便，这里只按照上界去划分，即这里将第 i 个桶的范围看做 `(i-1 桶的上界，i 桶的上界]`。特别的，对于最后一个桶，将其的上界视为无穷大。比方说一个直方图包含 ３ 个桶，范围分别是: [2，5]，[8，8]，[10，13]，查询的范围是 (3，20]，那么最终切分得到的查询范围就是 (3，5]，(5，8]，(8，20]。

将查询范围切分好后，会被存放在 [QueryFeedback](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L49) 中，以便在每个 Region 的结果返回时，调用 [Update](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L165) 函数来更新每个范围所包含的 key 数目。注意到这个函数需要两个参数：每个 Region 上扫描的 start key 以及 Region 上每一个扫描范围输出的 key 数目 output counts，那么要如何更新 `QueryFeedback` 中每个范围包含的 key 的数目呢？

继续以划分好的 (3，5]，(5，8]，(8，20] 为例，假设这个请求需要发送到两个 region 上，region1 的范围是 [0，6)，region2 的范围是 [6，30)，由于 coprocessor 在发请求的时候还会根据 Region 的范围切分 range，因此 region1 的请求范围是 (3，5]，(5，6)，region2 的请求范围是 [6，8]，(8，20]。为了将对应的 key 数目更新到 [QueryFeedback](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L49) 中，需要知道每一个 output count 对应的查询范围。注意到 coprocessor 返回的 output counts 其对应的 Range 都是连续的，并且同一个值只会对应一个 range，那么我们只需要知道第一个 output count 所对应的 range，即只需要知道这次扫描的 start key 就可以了。举个例子，对于 region1 来说，start key 是 3，那么 output counts 对应的 range 就是 (3，5]，(5，8]，对 region2 来说，start key 是 6，output countshangyipians 对应的 range 就是 (5，8]，(8，20]。

### 直方图的更新

在收集了 `QueryFeedback` 后，我们就可以去使用 [UpdateHistogram](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L536) 来更新直方图了。其大体上可以分为分裂与合并。

在 [splitBuckets](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L503) 中，我们实现了直方图的分裂：

* 首先，由于桶与桶之间的反馈信息不相关，为了方便，先将 `QueryFeedback` 用 [buildBucketFeedback](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L504) 拆分了每一个桶的反馈信息，并存放在 [BucketFeedback](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L213) 中。

* 接着，使用 [getSplitCount](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L507) 来根据可用的桶的个数和反馈信息的总数来决定分裂的数目。

* 对于每一个桶，将可以分裂的桶按照反馈信息数目的比例均分，然后用 [splitBucket](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L312) 来分裂出需要的桶的数目：

* 首先，[getBoundaries](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L312) 会每隔几个点取一个作为边界，得到新的桶。

* 然后，对于每一个桶，[refineBucketCount](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L32%201) 用与新生成的桶重合部分最多的反馈信息更新桶的深度。

值得注意的是，在分裂的时候，如果一个桶过小，那么这个桶不会被分裂；如果一个分裂后生成的桶过小，那么它也不会被生成。

在桶的分裂完成后，我们会使用 [mergeBuckets](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L467) 来合并桶，对于那些超过：

* 在分裂的时候，会记录每一个桶是不是新生成的，这样，对于原先就存在的桶，用 [getBucketScore](https://github.com/lamxTyler/tidb/blob/source-code/statistics/feedback.go#L476) 计算合并的之后产生的误差，令第一个桶占合并后桶的比例为 r，那么令合并后产生的误差为 abs（合并前第一个桶的高度 - r * 两个桶的高度和）/ 合并前第一个桶的高度。

* 接着，对每一桶的合并的误差进行排序。

* 最后，按照合并的误差从下到大的顺序，合并需要的桶。

## 统计信息使用

在查询语句中，我们常常会使用一些过滤条件，而统计信息估算的主要作用就是估计经过这些过滤条件后的数据条数，以便优化器选择最优的执行计划。

由于在单列上的查询比较简单，这里不再赘述，代码基本是按照 [统计信息（上）](https://pingcap.com/blog-cn/tidb-source-code-reading-12/) 中的原理实现，感兴趣可以参考 [histogram.go/lessRowCount](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L408)  以及 [cmsketch.go/queryValue](https://github.com/lamxTyler/tidb/blob/source-code/statistics/cmsketch.go#L69)。

### 多列查询

统计信息（上）中提到，[Selectivity](https://github.com/lamxTyler/tidb/blob/source-code/statistics/selectivity.go#L148) 是统计信息模块对优化器提供的最重要的接口，处理了多列查询的情况。Selectivity 的一个最重要的任务就是将所有的查询条件分成尽量少的组，使得每一组中的条件都可以用某一列或者某一索引上的统计信息进行估计，这样我们就可以做尽量少的独立性假设。

需要注意的是，我们将单列的统计信息分为 3 类：[indexType](https://github.com/lamxTyler/tidb/blob/source-code/statistics/selectivity.go#L42) 即索引列，[pkType](https://github.com/lamxTyler/tidb/blob/source-code/statistics/selectivity.go#L43) 即 Int 类型的主键，[colType](https://github.com/lamxTyler/tidb/blob/source-code/statistics/selectivity.go#L44) 即普通的列类型，如果一个条件可以同时被多种类型的统计信息覆盖，那么我们优先会选择 pkType 或者 indexType。

在 Selectivity 中，有如下几个步骤：

* [getMaskAndRange](https://github.com/lamxTyler/tidb/blob/source-code/statistics/selectivity.go#L230) 为每一列和每一个索引计算了可以覆盖的过滤条件，用一个 int64 来当做一个 bitset，并把将该列可以覆盖的过滤条件的位置置为 1。

* 接下来在 [getUsableSetsByGreedy](https://github.com/lamxTyler/tidb/blob/source-code/statistics/selectivity.go#L258) 中，选择尽量少的 bitset，来覆盖尽量多的过滤条件。每一次在还没有使用的 bitset 中，选择一个可以覆盖最多尚未覆盖的过滤条件。并且如果可以覆盖同样多的过滤条件，我们会优先选择 pkType 或者 indexType。

* 用统计信息（上）提到的方法对每一个列和每一个索引上的统计信息进行估计，并用独立性假设将它们组合起来当做最终的结果。

## 总结

统计信息的收集和维护是数据库的核心功能，对于基于代价的查询优化器，统计信息的准确性直接影响了查询效率。在分布式数据库中，收集统计信息和单机差别不大，但是维护统计信息有比较大的挑战，比如怎样在多节点更新的情况下，准确及时的维护统计信息。

对于直方图的动态更新，业界一般有两种方法：

* 对于每一次增删，都去更新对应的桶深。在一个桶的桶深过高的时候分裂桶，一般是把桶的宽度等分，不过这样很难准确的确定分界点，引起误差。

* 使用查询得到的真实数去反馈调整直方图，假定所有桶贡献的误差都是均匀的，用连续值假设去调整所有涉及到的桶。然而误差均匀的假设常常会引起问题，比如当新插入的值大于直方图的最大值时，就会把新插入的值引起的误差分摊到直方图中，从而引起误差。

目前 TiDB 的统计信息还是以单列的统计信息为主，为了减少独立性假设的使用，在将来 TiDB 会探索多列统计信息的收集和维护，为优化器提供更准确的统计信息。

