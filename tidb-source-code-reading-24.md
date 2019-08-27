---
title: TiDB 源码阅读系列文章（二十四）TiDB Binlog 源码解析
author: ['姚维']
date: 2019-01-15
summary: 本文将为大家介绍 TiDB 在执行 DML/DDL 语句过程中，如何将 binlog 数据发送给 TiDB Binlog 集群的 Pump 组件。
tags: ['TiDB 源码阅读','社区']
---


## TiDB Binlog Overview

这篇文章不是讲 TiDB Binlog 组件的源码，而是讲 TiDB 在执行 DML/DDL 语句过程中，如何将 binlog 数据 发送给 TiDB Binlog 集群的 Pump 组件。目前 TiDB 在 DML 上的 binlog 用的类似 [Row-based](https://dev.mysql.com/doc/refman/5.7/en/binary-log-formats.html) 的格式。TiDB Binlog 具体的架构细节可以参看这篇 [TiDB Ecosystem Tools 原理解读系列（一）：TiDB-Binlog 架构演进与实现原理](https://pingcap.com/blog-cn/tidb-ecosystem-tools-1/)。

**这里只描述 TiDB 中的代码实现。**

## DML binlog

TiDB 采用 protobuf 来编码 binlog，具体的格式可以见 [binlog.proto](https://github.com/pingcap/tipb/blob/master/proto/binlog/binlog.proto)。这里讨论 TiDB 写 binlog 的机制，以及 binlog 对 TiDB 写入的影响。

TiDB 会在 DML 语句提交，以及 DDL 语句完成的时候，向 pump 输出 binlog。

### Statement 执行阶段

DML 语句包括 Insert/Replace、Update、Delete，这里挑 Insert 语句来阐述，其他的语句行为都类似。首先在 Insert 语句执行完插入（未提交）之前，会把自己新增的数据记录在 `binlog.TableMutation` 结构体中。

```
// TableMutation 存储表中数据的变化
message TableMutation {
	    // 表的 id，唯一标识一个表
	    optional int64 table_id      = 1 [(gogoproto.nullable) = false]; 
	    
	    // 保存插入的每行数据
	    repeated bytes inserted_rows = 2;
	    
	    // 保存修改前和修改后的每行的数据
	    repeated bytes updated_rows  = 3;
	    
	    // 已废弃
	    repeated int64 deleted_ids   = 4;
	    
	    // 已废弃
	    repeated bytes deleted_pks   = 5;
	     
	    // 删除行的数据
	    repeated bytes deleted_rows  = 6;
	    
	    // 记录数据变更的顺序
	    repeated MutationType sequence = 7;
}
```


这个结构体保存于跟每个 Session 链接相关的事务上下文结构体中 `TxnState.mutations`。 一张表对应一个 `TableMutation` 对象，`TableMutation` 里面保存了这个事务对这张表的所有变更数据。Insert 会把当前语句插入的行，根据 `RowID` + `Row-value` 的格式编码之后，追加到 `TableMutation.InsertedRows` 中：

```
func (t *Table) addInsertBinlog(ctx context.Context, h int64, row []types.Datum, colIDs []int64) error {
	mutation := t.getMutation(ctx)
	pk, err := codec.EncodeValue(ctx.GetSessionVars().StmtCtx, nil, types.NewIntDatum(h))
	if err != nil {
		return errors.Trace(err)
	}
	value, err := tablecodec.EncodeRow(ctx.GetSessionVars().StmtCtx, row, colIDs, nil, nil)
	if err != nil {
		return errors.Trace(err)
	}
	bin := append(pk, value...)
	mutation.InsertedRows = append(mutation.InsertedRows, bin)
	mutation.Sequence = append(mutation.Sequence, binlog.MutationType_Insert)
	return nil
}
```

等到所有的语句都执行完之后，在 `TxnState.mutations` 中就保存了当前事务对所有表的变更数据。

### Commit 阶段

对于 DML 而言，TiDB 的事务采用 2-phase-commit 算法，一次事务提交会分为 Prewrite 阶段，以及 Commit 阶段。这里分两个阶段来看看 TiDB 具体的行为。

#### Prewrite binlog

在 `session.doCommit` 函数中，TiDB 会构造 `binlog.PrewriteValue`：

```
message PrewriteValue {
    optional int64         schema_version = 1 [(gogoproto.nullable) = false];
    repeated TableMutation mutations      = 2 [(gogoproto.nullable) = false];
}
```

这个 `PrewriteValue` 中包含了跟这次变动相关的所有行数据，TiDB 会填充一个类型为 `binlog.BinlogType_Prewrite` 的 Binlog：

```
info := &binloginfo.BinlogInfo{
	Data: &binlog.Binlog{
		Tp:            binlog.BinlogType_Prewrite,
		PrewriteValue: prewriteData,
	},
	Client: s.sessionVars.BinlogClient.(binlog.PumpClient),
}
```

TiDB 这里用一个事务的 Option `kv.BinlogInfo` 来把 `BinlogInfo` 绑定到当前要提交的 transaction 对象中：

```
s.txn.SetOption(kv.BinlogInfo, info)
```

在 `twoPhaseCommitter.execute` 中，在把数据 prewrite 到 TiKV 的同时，会调用 `twoPhaseCommitter.prewriteBinlog`，这里会把关联的 `binloginfo.BinlogInfo` 取出来，把 binlog 的 `binlog.PrewriteValue` 输出到 Pump。

```
binlogChan := c.prewriteBinlog()
err := c.prewriteKeys(NewBackoffer(prewriteMaxBackoff, ctx), c.keys)
if binlogChan != nil {
	binlogErr := <-binlogChan // 等待 write prewrite binlog 完成
	if binlogErr != nil {
		return errors.Trace(binlogErr)
	}
}
```

这里值得注意的是，在 prewrite 阶段，是需要等待 write prewrite binlog 完成之后，才能继续做接下去的提交的，这里是为了保证 TiDB 成功提交的事务，Pump 至少一定能收到 Prewrite binlog。

#### Commit binlog

在 `twoPhaseCommitter.execute` 事务提交结束之后，事务可能提交成功，也可能提交失败。TiDB 需要把这个状态告知 Pump：

```
err = committer.execute(ctx)
if err != nil {
	committer.writeFinishBinlog(binlog.BinlogType_Rollback, 0)
	return errors.Trace(err)
}
committer.writeFinishBinlog(binlog.BinlogType_Commit, int64(committer.commitTS))
```

如果发生了 error，那么输出的 binlog 类型就为 `binlog.BinlogType_Rollback`，如果成功提交，那么输出的 binlog 类型就为 `binlog.BinlogType_Commit`。

```
func (c *twoPhaseCommitter) writeFinishBinlog(tp binlog.BinlogType, commitTS int64) {
	if !c.shouldWriteBinlog() {
		return
	}
	binInfo := c.txn.us.GetOption(kv.BinlogInfo).(*binloginfo.BinlogInfo)
	binInfo.Data.Tp = tp
	binInfo.Data.CommitTs = commitTS
	go func() {
		err := binInfo.WriteBinlog(c.store.clusterID)
		if err != nil {
			log.Errorf("failed to write binlog: %v", err)
		}
	}()
}
```

值得注意的是，这里 WriteBinlog 是单独启动 goroutine 异步完成的，也就是 Commit 阶段，是不再需要等待写 binlog 完成的。这里可以节省一点 commit 的等待时间，这里不需要等待是因为 Pump 即使接收不到这个 Commit binlog，在超过 timeout 时间后，Pump 会自行根据 Prewrite binlog 到 TiKV 中确认当条事务的提交状态。

## DDL binlog

一个 DDL 有如下几个状态：

```
const (
	JobStateNone    		JobState = 0
	JobStateRunning 		JobState = 1
	JobStateRollingback  	JobState = 2
	JobStateRollbackDone 	JobState = 3
	JobStateDone         	JobState = 4
	JobStateSynced 			JobState = 6
	JobStateCancelling 		JobState = 7
)
```

这些状态代表了一个 DDL 任务所处的状态：

1. `JobStateNone`，代表 DDL 任务还在处理队列，TiDB 还没有开始做这个 DDL。

2. `JobStateRunning`，当 DDL Owner 开始处理这个任务的时候，会把状态设置为 `JobStateRunning`，之后 DDL 会开始变更，TiDB 的 Schema 可能会涉及多个状态的变更，这中间不会改变 DDL job 的状态，只会变更 Schema 的状态。

3. `JobStateDone`， 当 TiDB 完成自己所有的 Schema 状态变更之后，会把 Job 的状态改为 Done。

4. `JobStateSynced`，当 TiDB 每做一次 schema 状态变更，就会需要跟集群中的其他 TiDB 做一次同步，但是当 Job 状态为 `JobStateDone` 之后，在 TiDB 等到所有的 TiDB 节点同步之后，会将状态修改为 `JobStateSynced`。

5. `JobStateCancelling`，TiDB 提供语法 `ADMIN CANCEL DDL JOBS job_ids` 用于取消某个正在执行或者还未执行的 DDL 任务，当成功执行这个命令之后，DDL 任务的状态会变为 `JobStateCancelling`。

6. `JobStateRollingback`，当 DDL Owner 发现 Job 的状态变为 `JobStateCancelling` 之后，它会将 job 的状态改变为 `JobStateRollingback`，以示已经开始处理 cancel 请求。

7. `JobStateRollbackDone`，在做 cancel 的过程，也会涉及 Schema 状态的变更，也需要经历 Schema 的同步，等到状态回滚已经做完了，TiDB 会将 Job 的状态设置为 `JobStateRollbackDone`。

对于 binlog 而言，DDL 的 binlog 输出机制，跟 DML 语句也是类似的，只有开始处理事务提交阶段，才会开始写 binlog 出去。那么对于 DDL 来说，跟 DML 不一样，DML 有事务的概念，对于 DDL 来说，SQL 的事务是不影响 DDL 语句的。但是 DDL 里面，上面提到的 Job 的状态变更，是作为一个事务来提交的（保证状态一致性）。所以在每个状态变更，都会有一个事务与之对应，但是上面提到的中间状态，DDL 并不会往外写 binlog，只有 `JobStateRollbackDone` 以及 `JobStateDone` 这两种状态，TiDB 会认为 DDL 语句已经完成，会对外发送 binlog，发送之前，会把 Job 的状态从 `JobStateDone` 修改为 `JobStateSynced`，这次修改，也涉及一次事务提交。这块逻辑的代码如下：

```
worker.handleDDLJobQueue():

if job.IsDone() || job.IsRollbackDone() {
		binloginfo.SetDDLBinlog(d.binlogCli, txn, job.ID, job.Query)
		if !job.IsRollbackDone() {
			job.State = model.JobStateSynced
		}
		err = w.finishDDLJob(t, job)
		return errors.Trace(err)
}

type Binlog struct {
	DdlQuery []byte
	DdlJobId         int64
}
```


`DdlQuery` 会设置为原始的 DDL 语句，`DdlJobId` 会设置为 DDL 的任务 ID。

对于最后一次 Job 状态的提交，会有两条 binlog 与之对应，这里有几种情况：

1. 如果事务提交成功，类型分别为 `binlog.BinlogType_Prewrite` 和 `binlog.BinlogType_Commit`。

2. 如果事务提交失败，类型分别为 `binlog.BinlogType_Prewrite` 和 `binlog.BinlogType_Rollback`。

所以，Pumps 收到的 DDL binlog，如果类型为 `binlog.BinlogType_Rollback` 应该只认为如下状态是合法的：

1. `JobStateDone` （因为修改为 `JobStateSynced` 还未成功）

2. `JobStateRollbackDone`

如果类型为 `binlog.BinlogType_Commit`，应该只认为如下状态是合法的：

1. `JobStateSynced`

2. `JobStateRollbackDone`

当 TiDB 在提交最后一个 Job 状态的时候，如果事务提交失败了，那么 TiDB Owner 会尝试继续修改这个 Job，直到成功。也就是对于同一个 `DdlJobId`，后续还可能会有多次 binlog，直到出现 `binlog.BinlogType_Commit`。
