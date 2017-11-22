---
title: Percolator 和 TiDB 事务算法
author: ['黄东旭']
date: 2016-11-22
summary: 本文先概括的讲一下 Google Percolator 的大致流程。Percolator 是 Google 的上一代分布式事务解决方案，构建在 BigTable 之上，在 Google 内部用于网页索引更新的业务。TiDB 的事务模型沿用了 Percolator 的事务模型。
tags: ['TiDB', 'Percolator', '事务']
---


本文先概括的讲一下 Google Percolator 的大致流程。Percolator 是 Google 的上一代分布式事务解决方案，构建在 BigTable 之上，在 Google 内部 用于网页索引更新的业务，原始的论文[在此](http://research.google.com/pubs/pub36726.html)。原理比较简单，总体来说就是一个经过优化的二阶段提交的实现，进行了一个二级锁的优化。TiDB 的事务模型沿用了 Percolator 的事务模型。
总体的流程如下：

### 读写事务

1) 事务提交前，在客户端 buffer 所有的 update/delete 操作。
2) Prewrite 阶段:

首先在所有行的写操作中选出一个作为 primary，其他的为 secondaries。

PrewritePrimary: 对 primaryRow 写入 L 列(上锁)，L 列中记录本次事务的开始时间戳。写入 L 列前会检查:

1. 是否已经有别的客户端已经上锁 (Locking)。
2. 是否在本次事务开始时间之后，检查 W 列，是否有更新 [startTs, +Inf) 的写操作已经提交 (Conflict)。

在这两种种情况下会返回事务冲突。否则，就成功上锁。将行的内容写入 row 中，时间戳设置为 startTs。

将 primaryRow 的锁上好了以后，进行 secondaries 的 prewrite 流程:

1. 类似 primaryRow 的上锁流程，只不过锁的内容为事务开始时间及 primaryRow 的 Lock 的信息。
2. 检查的事项同 primaryRow 的一致。

当锁成功写入后，写入 row，时间戳设置为 startTs。

3) 以上 Prewrite 流程任何一步发生错误，都会进行回滚：删除 Lock，删除版本为 startTs 的数据。

4) 当 Prewrite 完成以后，进入 Commit 阶段，当前时间戳为 commitTs，且 commitTs> startTs :

1. commit primary：写入 W 列新数据，时间戳为 commitTs，内容为 startTs，表明数据的最新版本是 startTs 对应的数据。
2. 删除L列。

如果 primary row 提交失败的话，全事务回滚，回滚逻辑同 prewrite。如果 commit primary 成功，则可以异步的 commit secondaries, 流程和 commit primary 一致， 失败了也无所谓。

### 事务中的读操作

1. 检查该行是否有 L 列，时间戳为 [0, startTs]，如果有，表示目前有其他事务正占用此行，如果这个锁已经超时则尝试清除，否则等待超时或者其他事务主动解锁。注意此时不能直接返回老版本的数据，否则会发生幻读的问题。
2. 读取至 startTs 时该行最新的数据，方法是：读取 W 列，时间戳为 [0, startTs], 获取这一列的值，转化成时间戳 t, 然后读取此列于 t 版本的数据内容。

由于锁是分两级的，primary 和 seconary，只要 primary 的行锁去掉，就表示该事务已经成功 提交，这样的好处是 secondary 的 commit 是可以异步进行的，只是在异步提交进行的过程中 ，如果此时有读请求，可能会需要做一下锁的清理工作。
