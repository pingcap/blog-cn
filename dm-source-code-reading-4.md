---
title: DM 源码阅读系列文章（四）dump/load 全量同步的实现
author: ['杨非']
date: 2019-04-26
summary: 本文将详细介绍 dump 和 load 两个数据同步处理单元的设计实现，重点关注数据同步处理单元 interface 的实现，数据导入并发模型的设计，以及导入任务在暂停或出现异常后如何恢复。
tags: ['DM 源码阅读','社区']
---

本文为 TiDB Data Migration 源码阅读系列文章的第四篇，[《DM 源码阅读系列文章（三）数据同步处理单元介绍》](https://pingcap.com/blog-cn/dm-source-code-reading-3/) 介绍了数据同步处理单元实现的功能，数据同步流程的运行逻辑以及数据同步处理单元的 interface 设计。本篇文章在此基础上展开，详细介绍 dump 和 load 两个数据同步处理单元的设计实现，重点关注数据同步处理单元 interface 的实现，数据导入并发模型的设计，以及导入任务在暂停或出现异常后如何恢复。

## dump 处理单元

dump 处理单元的代码位于 [github.com/pingcap/dm/mydumper](https://github.com/pingcap/dm/tree/master/mydumper) 包内，作用是从上游 MySQL 将表结构和数据导出到逻辑 SQL 文件，由于该处理单元总是运行在任务的第一个阶段（full 模式和 all 模式），该处理单元每次运行不依赖于其他处理单元的处理结果。另一方面，如果在 dump 运行过程中被强制终止（例如在 dmctl 中执行 pause-task 或者 stop-task），也不会记录已经 dump 数据的 checkpoint 等信息。不记录 checkpoint 是因为每次运行 Mydumper 从上游导出数据，上游的数据都可能发生变更，为了能得到一致的数据和 metadata 信息，每次恢复任务或重新运行任务时该处理单元会 [清理旧的数据目录](https://github.com/pingcap/dm/blob/092b5e4378ce42cf6c2488dd06498792190a091b/mydumper/mydumper.go#L68)，重新开始一次完整的数据 dump。

导出表结构和数据的逻辑并不是在 DM 内部直接实现，而是 [通过 `os/exec` 包调用外部 mydumper 二进制文件](https://github.com/pingcap/dm/blob/092b5e4378ce42cf6c2488dd06498792190a091b/mydumper/mydumper.go#L104) 来完成。在 Mydumper 内部，我们需要关注以下几个问题：

* 数据导出时的并发模型是如何实现的。

* no-locks, lock-all-tables, less-locking 等参数有怎样的功能。

* 库表黑白名单的实现方式。

### Mydumper 的实现细节

Mydumper 的一次完整的运行流程从主线程开始，主线程按照以下步骤执行：

1. 解析参数。

2. [创建到数据库的连接](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1076)。

3. 会根据 `no-locks` 选项进行一系列的备份安全策略，包括 [`long query guard`](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1253-L1292) 和 [`lock all tables or FLUSH TABLES WITH READ LOCK`](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1294-L1453)。

4. [`START TRANSACTION WITH CONSISTENT SNAPSHOT`](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1469)。

5. [记录 binlog 位点信息](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1496-L1503)。

6. [`less locking` 处理线程的初始化](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1508-L1519)。

7. [普通导出线程初始化](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1525-L1530)。

8. [如果配置了 `trx-consistency-only` 选项，执行 `UNLOCK TABLES /* trx-only */` 释放之前获取的表锁。](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1534-L1539)注意，如果开启该选项，是无法保证非 InnoDB 表导出数据的一致性。更多关于一致性读的细节可以参考 [MySQL 官方文档 Consistent Nonlocking Reads 部分](https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html)。

9. [根据配置规则（包括 --database, --tables-list 和 --regex 配置）读取需要导出的 schema 和表信息，并在这个过程中有区分的记录 innodb_tables 和 non_innodb_table](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1541-L1566)。

10. [为工作子线程创建任务，并将任务 push 到相关的工作队列](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1572-L1646)。

11. [如果没有配置 `no-locks` 和 `trx-consistency-only` 选项，执行 UNLOCK TABLES /* FTWRL */ 释放锁](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1648-L1654)。

12. [如果开启 `less-locking`，等待所有 `less locking` 子线程退出](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1663-L1668)。

13. [等待所有工作子线程退出](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1670-L1679)。

工作线程的并发控制包括了两个层面，一层是在不同表级别的并发，另一层是同一张表级别的并发。Mydumper 的主线程会将一次同步任务拆分为多个同步子任务，并将每个子任务分发给同一个异步队列 `conf.queue_less_locking/conf.queue`，工作子线程从队列中获取任务并执行。具体的子任务划分包括以下策略：

+ 开启 `less-locking` 选项的非 InnoDB 表的处理。
    - [先将所有 `non_innodb_table` 分为 `num_threads` 组，分组方式是遍历这些表，依此将遍历到的表加入到当前数据量最小的分组，尽量保证每个分组内的数据量相近](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1574-L1586)。
    - 上述得到的每个分组内会包含一个或多个非 InnoDB 表，如果配置了 `rows-per-file` 选项，会对每张表进行 `chunks` 估算，[对于每一张表，如果估算结果包含多个 chunks，会将子任务进一步按照 `chunks` 进行拆分，分发 `chunks` 数量个子任务](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L3033-L3046)，[如果没有 `chunks` 划分，分发为一个独立的子任务](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L3047-L3057)。
    - 注意，在该模式下，子任务会 [发送到 `queue_less_locking`](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L3059)，并在编号为 `num_threads` ~ 2 * `num_threads` 的子线程中处理任务。
        - `less_locking_threads` 任务执行完成之后，[主线程就会 UNLOCK TABLES /* FTWRL */ 释放锁](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1648-L1654)，这样有助于减少锁持有的时间。主线程根据 `conf.unlock_tables` 来判断非 InnoDB 表是否全部导出，[普通工作线程](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L639-L641) 或者 [queue_less_locking](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L787-L789) 工作线程每次处理完一个非 InnoDB 表任务都会根据 `non_innodb_table_counter` 和 `non_innodb_done` 两个变量判断是否还有没有导出结束的非 InnoDB 表，如果都已经导出结束，就会向异步队列 `conf.unlock_tables` 中发送一条数据，表示可以解锁全局锁。
        - 每个 `less_locking_threads` 处理非 InnoDB 表任务时，会先 [加表锁](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L771-L778)，导出数据，最后 [解锁表锁](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L803)。

+ 未开启 `less-locking` 选项的非 InnoDB 表的处理。
    - [遍历每一张非 InnoDB 表，同样对每张表进行 `chunks` 估算，如果包含多个 `chunks`，按照 chunks 个数分发同样的子任务数；如果没有划分 `chunks`，每张表分发一个子任务。所有的任务都分发到 conf->queue 队列。](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1606-L1614)

+ InnoDB 表的处理。
    - 与未开启 `less-locking` 选项的非 InnoDB 表的处理相同，同样是 [按照表分发子任务，如果有 `chunks` 子任务会进一步细分](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1616-L1620)。

从上述的并发模型可以看出 Mydumper 首先按照表进行同步任务拆分，对于同一张表，如果配置 `rows-per-file` 参数，会根据该参数和表行数将表划分为合适的 `chunks` 数，这即是同一张表内部的并发。具体表行数的估算和 `chunks` 划分的实现见 [`get_chunks_for_table`](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L1885-L2004) 函数。

需要注意目前 DM 在任务配置中指定的库表黑白名单功能只应用于 load 和 binlog replication 处理单元。如果在 dump 处理单元内使用库表黑白名单功能，需要在同步任务配置文件的 dump 处理单元配置提供 extra-args 参数，并指定 Mydumper 相关参数，包括 --database, --tables-list 和 --regex。Mydumper 使用 regex 过滤库表的实现参考 [`check_regex`](https://github.com/pingcap/mydumper/blob/9493dd752b9ea8804458e56a955e7f74960fa969/mydumper.c#L314-L338) 函数。

## load 处理单元

load 处理单元的代码位于 [github.com/pingcap/dm/loader](https://github.com/pingcap/dm/tree/master/loader) 包内，该处理单元在 dump 处理单元运行结束后运行，读取 dump 处理单元导出的 SQL 文件解析并在下游数据库执行逻辑 SQL。我们重点分析 `Init` 和 `Process` 两个 interface 的实现。

### Init 实现细节

该阶段进行一些初始化和清理操作，并不会开始同步任务，如果在该阶段运行中出现错误，会通过 [rollback 机制](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L356-L361) 清理资源，不需要调用 Close 函数。该阶段包含的初始化操作包括以下几点：

* [创建 `checkpoint`](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L363)，`checkpoint` 用于记录全量数据的导入进度和 load 处理单元暂停或异常终止后，恢复或重新开始任务时可以从断点处继续导入数据。

* 应用任务配置的数据同步规则，包括以下规则：
    * [初始化黑白名单](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L370)
    * [初始化表路有规则](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L380)
    * [初始化列值转换规则](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L385-L390)

### Process 实现细节

该阶段的工作流程也很直观，通过 [一个收发数据类型为 `*pb.ProcessError` 的 `channel` 接收运行过程中出现的错误，出错后通过 context 的 `CancelFunc` 强制结束处理单元运行](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L408-L422)。在核心的 [数据导入函数](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L485) 中，工作模型与 mydumper 类似，即在 [主线程中分发任务](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L507)，[有多个工作线程执行具体的数据导入任务](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L500-L503)。具体的工作细节如下：

+ 主线程会按照库，表的顺序读取创建库语句文件 `<db-name>-schema-create.sql` 和建表语句文件 `<db-name>.<table-name>-schema-create.sql`，并在下游执行 SQL 创建相对应的库和表。

+ [主线程读取 `checkpoint` 信息，结合数据文件信息创建 fileJob 随机分发任务给一个工作子线程](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L944-L1015)，fileJob 任务的结构如下所示	：
	
	```go
	type fileJob struct {
	   schema    string
	   table     string
	   dataFile  string
	   offset    int64 // 表示读取文件的起始 offset，如果没有 checkpoint 断点信息该值为 0
	   info      *tableInfo // 保存原库表，目标库表，列名，insert 语句 column 名字列表等信息
	}
	```

+ 在每个工作线程内部，有一个循环不断从自己 `fileJobQueue` 获取任务，每次获取任务后会对文件进行解析，并将解析后的结果分批次打包为 SQL 语句分发给线程内部的另外一个工作协程，该工作协程负责处理 SQL 语句的执行。工作流程的伪代码如下所示，完整的代码参考 [`func (w *Worker) run()`](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L114-L173)：

	```go
	// worker 工作线程内分发给内部工作协程的任务结构
	type dataJob struct {
	   sql         string // insert 语句, insert into <table> values (x, y, z), (x2, y2, z2), … (xn, yn, zn);
	   schema      string // 目标数据库
	   file        string // SQL 文件名
	   offset      int64 // 本次导入数据在 SQL 文件的偏移量
	   lastOffset  int64 // 上一次已导入数据对应 SQL 文件偏移量
	}
	
	// SQL 语句执行协程
	doJob := func() {
	   for {
	       select {
	       case <-ctx.Done():
	           return
	       case job := <-jobQueue:
	           sqls := []string{
	               fmt.Sprintf("USE `%s`;", job.schema), // 指定插入数据的 schema
	               job.sql,
	               checkpoint.GenSQL(job.file, job.offset), // 更新 checkpoint 的 SQL 语句
	           }
	           executeSQLInOneTransaction(sqls) // 在一个事务中执行上述 3 条 SQL 语句
	       }
	   }
	}
	​
	// worker 主线程
	for {
	   select {
	   case <-ctx.Done():
	       return
	   case job := <-fileJobQueue:
	       go doJob()
	       readDataFileAndDispatchSQLJobs(ctx, dir, job.dataFile, job.offset, job.info)
	   }
	}
	```

+ [`dispatchSQL`](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L192) 函数负责在工作线程内部读取 SQL 文件和重写 SQL，该函数会在运行初始阶段 [创建所操作表的 `checkpoint` 信息](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L211)，需要注意在任务中断恢复之后，如果这个文件的导入还没有完成，[`checkpoint.Init` 仍然会执行，但是这次运行不会更新该文件的 `checkpoint` 信息](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/checkpoint.go#L271-L274)。[列值转换和库表路由也是在这个阶段内完成](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L256-L264)。

    - 列值转换：需要对输入 SQL 进行解析拆分为每一个 field，对需要转换的 field 进行转换操作，然后重新拼接起 SQL 语句。详细重写流程见 [reassemble](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/convert_data.go#L293) 函数。

    - 库表路由：这种场景下只需要 [替换源表到目标表](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L263) 即可。

+ 在工作线程执行一个批次的 SQL 语句之前，[会首先根据文件 `offset` 信息生成一条更新 checkpoint 的语句，加入到打包的 SQL 语句中](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/loader.go#L132-L137)，具体执行时这些语句会 [在一个事务中提交](https://github.com/pingcap/dm/blob/25f95ee08d008fb6469f0b172e432270aaa6be52/loader/db.go#L152-L195)，这样就保证了断点信息的准确性，如果导入过程暂停或中断，恢复任务后从断点重新同步可以保证数据一致。

## 小结

本篇详细介绍 dump 和 load 两个数据同步处理单元的设计实现，对核心 interface 实现、数据导入并发模型、数据导入暂停或中断的恢复进行了分析。接下来的文章会继续介绍 `binlog replication`，`relay log` 两个数据同步处理单元的实现。
