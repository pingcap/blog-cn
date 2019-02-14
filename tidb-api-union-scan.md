---
title: TiDB 下推 API 实现细节 - Union Scan
author: ['周昱行']
date: 2016-06-18
summary: TiDB 集群的架构分为上层的 SQL 层和底层的 KV 层，SQL 层通过调用 KV 层的 API 读写数据，由于 SQL 层的节点和 KV 层节点通常不在一台机器上，所以，每次调用 KV 的 API 都是一次 RPC, 而往往一个普通的 Select 语句的执行，需要调用几十到几十万次 KV 的接口，这样的结果就是性能非常差，绝大部分时间都消耗在 RPC 上。为了解决这个问题，TiDB 实现了下推 API，把一部分简单的 SQL 层的执行逻辑下推到 KV 层执行，让 KV 层可以理解 Table 和 Column，可以批量读取多行结果，可以用 Where 里的 Expression 对结果进行过滤, 可以计算聚合函数，大幅减少了 RPC 次数和数据的传输量。
tags: ['TiDB', '分布式计算']
meetup_type: memoir
---

TiDB 集群的架构分为上层的 SQL 层和底层的 KV 层，SQL 层通过调用 KV 层的 API 读写数据，由于 SQL 层的节点和 KV 层节点通常不在一台机器上，所以，每次调用 KV 的 API 都是一次 RPC, 而往往一个普通的 `Select` 语句的执行，需要调用几十到几十万次 KV 的接口，这样的结果就是性能非常差，绝大部分时间都消耗在 RPC 上。

为了解决这个问题，TiDB 实现了下推 API，把一部分简单的 SQL 层的执行逻辑下推到 KV 层执行，让 KV 层可以理解 Table 和 Column，可以批量读取多行结果，可以用 `Where` 里的 Expression 对结果进行过滤, 可以计算聚合函数，大幅减少了 RPC 次数和数据的传输量。

TiDB 的下推 API 通过把 SQL 层的计算下推到 KV 层，大幅减少 RPC 次数和数据传输量，使性能得到数量级的提升。但是当我们一开始启用下推 API 的时候，发现了一个问题，就是当事务写入了数据，但是还未提交的时候，又执行了 `Select` 操作。

这个时候，刚刚写入的未提交的脏数据读不到，得到的结果是错误的，比如我们在一个空表 t 执行：

``` sql
begin;
insert t values (1);
select * from t;
```

这时我们期待的结果是一条记录 “1”，但是启用下推 API 后得到的结果是空。

导致这个问题的原因是我们的事务在提交之前，写入的数据是 buffer 在 SQL 层，并没有写入 KV, 而下推 API 直接从 KV 读取数据，得到的结果直接返回，所以得到了空的结果。

但是既然 KV 层读取不到未提交的脏数据，那在启用下推 API 之前，是如何得到正确结果的呢？

这就涉及到 SQL 层的 Buffer 实现。当初为了解决未提交事务的 Buffer 可见性问题，SQL 层实现了一个 UnionStore 的结构，UnionStore 对 Buffer 和 KV 层接口做了一个封装，事务对 KV 的读写都经过 UnionStore，当 UnionStore 遇到读请求时，会先在 Buffer 里找，Buffer 找不到时，才会调用 KV 层的接口，读取 KV 层的数据。所以相当于把 Buffer 和 KV 的数据做了一个 Merge，返回 Merge 后的正确结果。Buffer 的数据是用 goleveldb 的 MemDB 存储的，所以是有序的，当需要遍历数据的时候，UnionStore 会同时创建 Buffer 的 Iterator 和 KV 的 Iterator，遍历的算法类似 LevelDB，把两个 Iterator merge 成一个。

UnionStore 的实现是基于 Key Value 的，但是下推 API 返回的结果是基于 Row 的，也就是说，我们虽然有脏数据 Buffer 和下推 API 返回的结果集, 但是我们没有办法把这两部分数据合并在一起, 所以我们为了绕过这个问题，加了一个判断条件，当事务写入了 Buffer，包含了脏数据以后，就不走下推 API，而是使用基础的 KV API。

在我们刚刚开始启用下推 API 的时候，因为性能基准比较低，而且带脏数据的下推请求只占很小的一部分，所以我选择暂时绕过这个问题。

但是当全面启用下推 API 以后，整体性能已经大幅提升，这时带脏数据的请求无法走下推 API 这个 worst case 问题就渐渐凸显出来。

比如说，我们如果需要在一个事务里 `UPDATE` 多个行，就一定会遇到下推 API 无法使用，降级到基础 KV API 的问题。

假设我们创建一个表，插入了两行数据：

```sql
create table t (c int);
insert t values (1), (4);
```

这时我们执行这样一个事务：

```sql
begin;
update t set c = 2 where c = 1;
update t set c = 3 where c = 4;
```

`UPDATE` 语句执行的过程分两步，第一步是先读取到需要更新的数据，第二步把更新的数据写入 Buffer。

也就是 `UPDATE` 包含了一次 `SELECT` 请求。当第一个 `UPDATE` 语句执行的时候，因为没有脏数据，所以读请求会走下推 API，但是第一个 `UPDATE` 语句执行完后，事务就有了脏数据，再执行第二个 `UPDATE` 的时候，无法使用下推 API, 会导致性能大幅下降。

解决这个问题的方案，最容易想到的是在 KV 层实现 UnionStore 相同的算法，当发送下推 API 请求时，把 Buffer 一并传下去。

但是这个方案的缺点也很明显，就是计算和存储不在同一节点，不符合就近计算原则。脏数据是在 SQL 层生成并存储的，本来应该在 SQL 进行 Merge，但是却要传输到 KV 层去 Merge，如果 Buffer 的数据很多，传输 Buffer 带来的开销就会很大。

最终我们设计实现了一个更好的方案 Union Scan，在不需要把 Buffer 传输到 KV 层，不修改 KV 层的情况下，解决了脏数据的可见性问题。

### 下面是这个算法的简介

脏数据缓存在 SQL 层，要让它可见，一定是需要 Merge 的，当我们使用下推 API, 只拿到了一堆 Row，这时怎么 Merge 呢？

如果我们不做 Merge，直接返回给用户结果集，错误表现的就是少了某些 row，多了某些 row，或某些 row 的数据是旧的。

如果我们把 `INSERT`, `UPDATE`, `DELETE` 的修改操作，以 row 为单位记录下来，这样和下推 API 返回的结果就是同样的形式了，就可以很方便的做 Merge 的计算了。

所以 Union Scan 的算法就是以 Row 为单位，把事务的修改操作保存起来，最终和下推 API 返回的结果集进行 Merge，返回给客户端。

我们为每个事务在对某个 table 执行写操作时，创建一个 dirtyTable 保存这个事务的修改，dirtyTable 包含两个 map，一个是 addedRows，用来保存新写入的 row，另一个是 removedRows，用来保存删除的 row，对于不同的操作，我们需要对这两个 map 做不同的操作。

对于 `INSERT`，我们需要把 row 添加到 addedRows 里。

对于 `DELETE`，我们需要把 row 从 addedRows 里删掉，然后把 row 添加到 removedRows 里。

对于 `UPDATE`，相当于先执行 `DELETE`, 再执行 `INSERT。`

当我们从下推 API 得到了结果集之后，我们下面把它叫做快照结果集，Merge 的算法如下：

对于每一条快照结果集里的 Row，在 removedRows 里查找，如果有，那么代表这一条结果已经被删掉，那么把它从结果集里删掉，得到过滤后的结果集。

把 addedRows 里的所有 Row，放到一个 slice 里，并对这个 slice 用快照结果集相同的顺序排序，生成脏数据结果集。

返回结果的时候，将过滤后的快照结果集与脏数据结果集进行 Merge。

实现了 Union Scan 以后，所有的读请求都可以使用下推 API 加速，大幅提升了 worst case 的性能。
