---
title: TiDB 源码阅读系列文章（十六）INSERT 语句详解
author: ['于帅鹏']
date: 2018-08-17
summary: 本文将首先介绍在 TiDB 中的 INSERT 语句的分类，以及各语句的语法和语义，然后分别介绍五种 INSERT 语句的源码实现，enjoy~
tags: ['源码阅读','TiDB','社区']
---


在之前的一篇文章 [《TiDB 源码阅读系列文章（四）INSERT 语句概览》](https://pingcap.com/blog-cn/tidb-source-code-reading-4) 中，我们已经介绍了 INSERT 语句的大体流程。为什么需要为 INSERT 单独再写一篇？因为在 TiDB 中，单纯插入一条数据是最简单的情况，也是最常用的情况；更为复杂的是在 INSERT 语句中设定各种行为，比如，对于 Unique Key 冲突的情况应如何处理：是报错？是忽略当前插入的数据？还是覆盖已有数据？所以，这篇会为大家继续深入介绍 INSERT 语句。

本文将首先介绍在 TiDB 中的 INSERT 语句的分类，以及各语句的语法和语义，然后分别介绍五种 INSERT 语句的源码实现。

# INSERT 语句的种类

从广义上讲，TiDB 有以下六种 INSERT 语句：

* `Basic INSERT`

* `INSERT IGNORE`

* `INSERT ON DUPLICATE KEY UPDATE`

* `INSERT IGNORE ON DUPLICATE KEY UPDATE`

* `REPLACE`

* `LOAD DATA`

这六种语句理论上都属于 INSERT 语句。

第一种，`Basic INSERT`，即是最普通的 INSERT 语句，语法 `INSERT INTO VALUES ()`，语义为插入一条语句，若发生唯一约束冲突（主键冲突、唯一索引冲突），则返回执行失败。

第二种，语法 `INSERT IGNORE INTO VALUES ()`，是当 INSERT 的时候遇到唯一约束冲突后，忽略当前 INSERT 的行，并记一个 warning。当语句执行结束后，可以通过 `SHOW WARNINGS` 看到哪些行没有被插入。

第三种，语法 `INSERT INTO VALUES () ON DUPLICATE KEY UPDATE`，是当冲突后，更新冲突行后插入数据。如果更新后的行跟表中另一行冲突，则返回错误。

第四种，是在上一种情况，更新后的行又跟另一行冲突后，不插入该行并显示为一个 warning。

第五种，语法 `REPLACE INTO VALUES ()`，是当冲突后，删除表上的冲突行，并继续尝试插入数据，如再次冲突，则继续删除标上冲突数据，直到表上没有与改行冲突的数据后，插入数据。

最后一种，语法 `LOAD DATA INFILE INTO` 的语义与 `INSERT IGNORE` 相同，都是冲突即忽略，不同的是 `LOAD DATA` 的作用是将数据文件导入到表中，也就是其数据来源于 csv 数据文件。

由于 `INSERT IGNORE ON DUPLICATE KEY UPDATE` 是在 `INSERT ON DUPLICATE KEY UPDATE` 上做了些特殊处理，将不再单独详细介绍，而是放在同一小节中介绍；`LOAD DATA` 由于其自身的特殊性，将留到其他篇章介绍。


# Basic INSERT 语句

几种 INSERT 语句的最大不同在于执行层面，这里接着 [《TiDB 源码阅读系列文章（四）INSERT 语句概览》](https://pingcap.com/blog-cn/tidb-source-code-reading-4) 来讲语句执行过程。不记得前面内容的同学可以返回去看原文章。

INSERT 的执行逻辑在 [executor/insert.go](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert.go) 中。其实前面讲的前四种 INSERT 的执行逻辑都在这个文件里。这里先讲最普通的 `Basic INSERT`。

`InsertExec` 是 INSERT 的执行器实现，其实现了 Executor 接口。最重要的是下面三个接口：

* Open：进行一些初始化

* Next：执行写入操作

* Close：做一些清理工作

其中最重要也是最复杂的是 Next 方法，根据是否通过一个 SELECT 语句来获取数据（`INSERT SELECT FROM`），将 Next 流程分为，[insertRows](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L180:24) 和 [insertRowsFromSelect](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L277:24) 两个流程。两个流程最终都会进入 `exec` 函数，执行 INSERT。

`exec` 函数里处理了前四种 INSERT 语句，其中本节要讲的普通 INSERT 直接进入了 [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22)。

在讲 [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22) 之前，我们先看一段 SQL。

```sql
CREATE TABLE t (i INT UNIQUE);
INSERT INTO t VALUES (1);
BEGIN;
INSERT INTO t VALUES (1);
COMMIT;
```

把这段 SQL 分别一行行地粘在 MySQL 和 TiDB 中看下结果。

MySQL：

```sql
mysql> CREATE TABLE t (i INT UNIQUE);
Query OK, 0 rows affected (0.15 sec)

mysql> INSERT INTO t VALUES (1);
Query OK, 1 row affected (0.01 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO t VALUES (1);
ERROR 1062 (23000): Duplicate entry '1' for key 'i'
mysql> COMMIT;
Query OK, 0 rows affected (0.11 sec)
```

TiDB：

```sql
mysql> CREATE TABLE t (i INT UNIQUE);
Query OK, 0 rows affected (1.04 sec)

mysql> INSERT INTO t VALUES (1);
Query OK, 1 row affected (0.12 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO t VALUES (1);
Query OK, 1 row affected (0.00 sec)

mysql> COMMIT;
ERROR 1062 (23000): Duplicate entry '1' for key 'i'
```

可以看出来，对于 INSERT 语句 TiDB 是在事务提交的时候才做冲突检测而 MySQL 是在语句执行的时候做的检测。这样处理的原因是，TiDB 在设计上，与 TiKV 是分层的结构，为了保证高效率的执行，在事务内只有读操作是必须从存储引擎获取数据，而所有的写操作都事先放在单 TiDB 实例内事务自有的 [memDbBuffer](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/kv/memdb_buffer.go#L31) 中，在事务提交时才一次性将事务写入 TiKV。在实现中是在 [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22) 中设置了 [PresumeKeyNotExists](https://github.com/pingcap/tidb/blob/e28a81813cfd290296df32056d437ccd17f321fe/kv/kv.go#L23) 选项，所有的 INSERT 操作如果在本地检测没发现冲突，就先假设插入不会发生冲突，不需要去 TiKV 中检查冲突数据是否存在，只将这些数据标记为待检测状态。最后到提交过程中，统一将整个事务里待检测数据使用 `BatchGet` 接口做一次批量检测。

当所有的数据都通过 [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22) 执行完插入后，INSERT 语句基本结束，剩余的工作为设置一下 lastInsertID 等返回信息，并最终将其结果返回给客户端。

# INSERT IGNORE 语句

`INSERT IGNORE` 的语义在前面已经介绍了。之前介绍了普通 INSERT 在提交的时候才检查，那 `INSERT IGNORE` 是否可以呢？答案是不行的。因为：

1. `INSERT IGNORE` 如果在提交时检测，那事务模块就需要知道哪些行需要忽略，哪些直接报错回滚，这无疑增加了模块间的耦合。

2. 用户希望立刻获取 `INSERT IGNORE` 有哪些行没有写入进去。即，立刻通过 `SHOW WARNINGS` 看到哪些行实际没有写入。

这就需要在执行 `INSERT IGNORE` 的时候，及时检查数据的冲突情况。一个显而易见的做法是，把需要插入的数据试着读出来，当发现冲突后，记一个 warning，再继续下一行。但是对于一个语句插入多行的情况，就需要反复从 TiKV 读取数据来进行检测，显然，这样的效率并不高。于是，TiDB 实现了 [batchChecker](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L43:6)，代码在 [executor/batch_checker.go](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/batch_checker.go)。

在 [batchChecker](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L43:6) 中，首先，拿待插入的数据，将其中可能冲突的唯一约束在 [getKeysNeedCheck](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L85:24) 中构造成 Key（TiDB 是通过构造唯一的 Key 来实现唯一约束的，详见 [《三篇文章了解 TiDB 技术内幕——说计算》](https://pingcap.com/blog-cn/tidb-internal-2/)）。

然后，将构造出来的 Key 通过 [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) 一次性读上来，得到一个 Key-Value map，能被读到的都是冲突的数据。

最后，拿即将插入的数据的 Key 到 [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) 的结果中进行查询。如果查到了冲突的行，构造好 warning 信息，然后开始下一行，如果查不到冲突的行，就可以进行安全的 INSERT 了。这部分的实现在 [batchCheckAndInsert](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L490:24) 中。

同样，在所有数据执行完插入后，设置返回信息，并将执行结果返回客户端。

# INSERT ON DUPLICATE KEY UPDATE 语句

`INSERT ON DUPLICATE KEY UPDATE` 是几种 INSERT 语句中最为复杂的。其语义的本质是包含了一个 INSERT 和 一个 UPDATE。较之与其他 INSERT 复杂的地方就在于，UPDATE 语义是可以将一行更新成任何合法的样子。

在上一节中，介绍了 TiDB 中对于特殊的 INSERT 语句采用了 batch 的方式来实现其冲突检查。在处理 `INSERT ON DUPLICATE KEY UPDATE` 的时候我们采用了同样的方式，但由于语义的复杂性，实现步骤也复杂了不少。

首先，与 `INSERT IGNORE` 相同，首先将待插入数据构造出来的 Key，通过 [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) 一次性地读出来，得到一个 Key-Value map。再把所有读出来的 Key 对应的表上的记录也通过一次 [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) 读出来，这部分数据是为了将来做 UPDATE 准备的，具体实现在 [initDupOldRowValue](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L225:24)。

然后，在做冲突检查的时候，如果遇到冲突，则首先进行一次 UPDATE。我们在前面 Basic INSERT 小节中已经介绍了，TiDB 的 INSERT 是提交的时候才去 TiKV 真正执行。同样的，UPDATE 语句也是在事务提交的时候才真正去 TiKV 执行的。在这次 UPDATE 中，可能还是会遇到唯一约束冲突的问题，如果遇到了，此时即报错返回，如果该语句是 `INSERT IGNORE ON DUPLICATE KEY UPDATE` 则会忽略这个错误，继续下一行。

在上一步的 UPDATE 中，还需要处理以下场景，如下面这个 SQL：

```sql
CREATE TABLE t (i INT UNIQUE);
INSERT INTO t VALUES (1), (1) ON DUPLICATE KEY UPDATE i = i;
```

可以看到，这个 SQL 中，表中原来并没有数据，第二句的 INSERT 也就不可能读到可能冲突的数据，但是，这句 INSERT 本身要插入的两行数据之间冲突了。这里的正确执行应该是，第一个 1 正常插入，第二个 1 插入的时候发现有冲突，更新第一个 1。此时，就需要做如下处理。将上一步被 UPDATE 的数据对应的 Key-Value 从第一步的 Key-Value map 中删掉，将 UPDATE 出来的数据再根据其表信息构造出唯一约束的 Key 和 Value，把这个 Key-Value 对放回第一步读出来 Key-Value map 中，用于后续数据进行冲突检查。这个细节的实现在 [fillBackKeys](https://github.com/pingcap/tidb/blob/2fba9931c7ffbb6dd939d5b890508eaa21281b4f/executor/batch_checker.go#L232)。这种场景同样出现在，其他 INSERT 语句中，如 `INSERT IGNORE`、`REPLACE`、`LOAD DATA`。之所以在这里介绍是因为，`INSERT ON DUPLICATE KEY UPDATE` 是最能完整展现 `batchChecker` 的各方面的语句。

最后，同样在所有数据执行完插入/更新后，设置返回信息，并将执行结果返回客户端。

# REPLACE 语句

REPLACE 语句虽然它看起来像是独立的一类 DML，实际上观察语法的话，它与 `Basic INSERT` 只是把 INSERT 换成了 REPLACE。与之前介绍的所有 INSERT 语句不同的是，REPLACE 语句是一个一对多的语句。简要说明一下就是，一般的 INSERT 语句如果需要 INSERT 某一行，那将会当遭遇了唯一约束冲突的时候，出现以下几种处理方式：

* 放弃插入，报错返回：`Basic INSERT`

* 放弃插入，不报错：`INSERT IGNORE`

* 放弃插入，改成更新冲突的行，如果更新的值再次冲突

* 报错：`INSERT ON DUPLICATE KEY UPDATE`

* 不报错：`INSERT IGNORE ON DUPLICATE KEY UPDATE`

他们都是处理一行数据跟表中的某一行冲突时的不同处理。但是 REPLACE 语句不同，它将会删除遇到的所有冲突行，直到没有冲突后再插入数据。如果表中有 5 个唯一索引，那有可能有 5 条与等待插入的行冲突的行。那么 REPLACE 语句将会一次性删除这 5 行，再将自己插入。看以下 SQL：

```sql
CREATE TABLE t (
i int unique, 
j int unique, 
k int unique, 
l int unique, 
m int unique);

INSERT INTO t VALUES 
(1, 1, 1, 1, 1), 
(2, 2, 2, 2, 2), 
(3, 3, 3, 3, 3), 
(4, 4, 4, 4, 4);

REPLACE INTO t VALUES (1, 2, 3, 4, 5);

SELECT * FROM t;
i j k l m
1 2 3 4 5
```

在执行完之后，实际影响了 5 行数据。

理解了 REPLACE 语句的特殊性以后，我们就可以更容易理解其具体实现。

与 INSERT 语句类似，REPLACE 语句的主要执行部分也在其 Next 方法中，与 INSERT 不同的是，其中的 [insertRowsFromSelect](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L277:24) 和 [insertRows](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L180:24) 传递了 [ReplaceExec](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L27) 自己的 [exec](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L160) 方法。在 [exec](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L160) 中调用了 [replaceRow](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L95)，其中同样使用了 [batchChecker](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L43:6) 中的批量冲突检测，与 INSERT 有所不同的是，这里会删除一切检测出的冲突，最后将待插入行写入。

# 写在最后

INSERT 语句是所有 DML 语句中最复杂，功能最强大多变的一个。其既有像 `INSERT ON DUPLICATE UPDATE` 这种能执行 INSERT 也能执行 UPDATE 的语句，也有像 REPLACE 这种一行数据能影响许多行数据的语句。INSERT 语句自身都可以连接一个 SELECT 语句作为待插入数据的输入，因此，其又受到了来自 planner 的影响（关于 planner 的部分详见相关的源码阅读文章： [（七）基于规则的优化](https://www.pingcap.com/blog-cn/tidb-source-code-reading-7/) 和 [（八）基于代价的优化](https://www.pingcap.com/blog-cn/tidb-source-code-reading-8/)）。熟悉 TiDB 的 INSERT 各个语句实现，可以帮助各位读者在将来使用这些语句时，更好地根据其特色使用最为合理、高效语句。另外，如果有兴趣向 TiDB 贡献代码的读者，也可以通过本文更快的理解这部分的实现。
