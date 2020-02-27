---
title: TiDB Binlog 源码阅读系列文章 （九）同步数据到下游
author: ['satoru']
date: 2020-02-26
summary: 同步数据这一步重要操作由 Drainer 模块支持，它可以将 binlog 同步到 TiDB / MySQL / Kafka / File （增量备份）等下游组件
tags: ['TiDB Binlog 源码阅读','社区']
---

[上篇文章](https://pingcap.com/blog-cn/tidb-binlog-source-code-reading-8/)介绍了用于将 binlog 同步到 MySQL / TiDB 的 Loader package，本文往回退一步，介绍 Drainer 同步到不同下游的机制。

[TiDB Binlog（github.com/pingcap/tidb-binlog）](http://mp.weixin.qq.com/s?__biz=MzI3NDIxNTQyOQ==&mid=2247487391&idx=1&sn=3e173b9c634e028824a69f67a506dd11&chksm=eb1628f5dc61a1e35fcbad1525857678de705b202a9d9765a71de8e79d2229cc5440686a10fc&scene=21#wechat_redirect)用于收集 TiDB 的 binlog，并准实时同步给下游。 同步数据这一步重要操作由 Drainer 模块支持，它可以将 binlog 同步到 TiDB / MySQL / Kafka / File （增量备份）等下游组件。

*   对于 TiDB 和 MySQL 两种类型的下游组件，Drainer 会从 binlog 中还原出对应的 SQL 操作在下游直接执行；

*   对于 Kafka 和 File（增量备份）两种类型的下游组件，输出约定编码格式的 binlog。用户可以定制后续各种处理流程，如更新搜索引擎索引、清除缓存、增量备份等。TiDB Binlog 自带工具 Reparo 实现了将增量备份数据（下游类型为 File（增量备份））同步到 TiDB / MySQL 的功能。

本文将按以下几个小节介绍 Drainer 如何将收到的 binlog 同步到下游：

1.  Drainer Sync 模块：Drainer 通过 `Sync` 模块调度整个同步过程，所有的下游相关的同步逻辑统一封装成了 `Syncer` 接口。

2.  恢复工具 Reparo （读音：reh-PAH-roh）：从下游保存的 File（增量备份）中读取 binlog 同步到 TiDB / MySQL。

## Drainer Sync 模块

### Syncer

同步机制的核心是 `Syncer` 接口，定义如下：

```
// Syncer sync binlog item to downstream
type Syncer interface {
  // Sync the binlog item to downstream
  Sync(item *Item) error
  // will be close if Close normally or meet error, call Error() to check it
  Successes() <-chan *Item
  // Return not nil if fail to sync data to downstream or nil if closed normally
  Error() <-chan error
  // Close the Syncer, no more item can be added by `Sync`
  Close() error
}

```

其中 `Sync` 方法表示异步地向下游同步一个 binlog，对应的参数类型是 *[Item](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/sync/syncer.go#L22-L27)，这是一个封装了 binlog 的结构体；`Successes` 方法返回一个 channel，从中可以读取已经成功同步到下游的 Item；`Error` 方法返回一个 channel，当 `Syncer` 同步过程出错中断时，会往这个 channel 发送遇到的错误；`Close` 用于关掉 `Syncer`，释放资源。

支持的每个下游类型在 drainer/sync 目录下都有一个对应的 Syncer 实现，例如 MySQL 对应的是 `mysql.go` 里的 [MySQLSyncer](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/sync/mysql.go#L30)，Kafka 对应的是 `kafka.go` 里的 [KafkaSyncer](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/sync/kafka.go#L36)。Drainer 启动时，会根据配置文件中指定的下游，[找到对应的 Syncer 实现](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/syncer.go#L91)，然后就可以用统一的接口管理整个同步过程了。

### Checkpoint

同步进程可能因为各种原因退出，重启后要恢复同步就需要知道上次同步的进度。在 Drainer 里记录同步进度的功能抽象成 `Checkpoint` 接口，其定义如下：

```
type CheckPoint interface {
  // Load loads checkpoint information.
  Load() error

  // Save saves checkpoint information.
  Save(int64) error

  // Pos gets position information.
  TS() int64

  // Close closes the CheckPoint and release resources, after closed other methods should not be called again.
  Close() error
}

```

从以上定义中可以看到，`Save` 的参数和 TS 的返回结果都是 int64 类型，因为同步的进度是以 TiDB 中单调递增的 commit timestamp 来记录的，它的类型就是 int64。

Drainer 支持不同类型的 Checkpoint 实现，例如  `mysql.go` 里的 `MySQLCheckpoint`，默认将 commit timestamp 写到 tidb_binlog 库下的 checkpoint 表。Drainer 会根据下游类型自动选择不同的 Checkpoint 实现，例如 TiDB / MySQL 的下游就会使用 [MySQLCheckPoint](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/checkpoint/mysql.go#L33)，File（增量备份） 则使用 [PbCheckpoint](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/checkpoint/pb.go#L27)。

在 Syncer 小节，我们看到 Syncer 的 `Successes` 方法提供了一个 channel 用来接收已经处理完毕的 binlog，收到 binlog 后，我们用 Checkpoint 的 `Save` 方法保存 binlog 的 commit timestamp 就可以记下同步进度，细节可查看源码中的 [handleSuccess](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/syncer.go#L180) 方法。

### Translator

Syncer 在收到 binlog 后需要将里面记录的变更转换成适合下游 Syncer 类型的格式，这部分实现在 [drainer/translator](https://github.com/pingcap/tidb-binlog/tree/v3.0.0/drainer/translator) 包。

以下游是 MySQL / TiDB 的情况为例。`MySQLSyncer.Sync` 会先调用 [TiBinlogToTxn](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/translator/mysql.go#L105)

将 binlog 转换成 loader.Txn 以便接入下层的 `loader` 模块 （loader 接收一个个 [loader.Txn](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/pkg/loader/model.go#L57) 结构并还原成对应的 SQL 批量写入 MySQL / TiDB）。

`loader.Txn` 定义如下：

```
// Txn holds transaction info, an DDL or DML sequences
type Txn struct {
  DMLs []*DML
  DDL  *DDL

  // This field is used to hold arbitrary data you wish to include so it
  // will be available when receiving on the Successes channel
  Metadata interface{}
}
```

Txn 主要有两类：DDL 和 DML。`Metadata` 目前放的就是传给 `Sync` 的 *Item 对象。DDL 的情况比较简单，因为 binlog 中已经直接包含了我们要用到的 DDL Query。DML 则需要遍历 binlog 中的一个个行变更，根据它的类型 insert / update / delete 还原成相应的 `loader.DML`。

### Schema

上个小节中，我们提到了对行变更数据的解析，在 binlog 中编码的行变更是没有列信息的，我们需要查到对应版本的列信息才能还原出 SQL 语义。Schema 就是解决这个问题的模块。

在 Drainer 启动时，会调用 [loadHistoryDDLJobs](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/server.go#L179) 从 TiKV 处查询截至当前时间所有已完成的 DDL Job 记录，按 `SchemaVersion` 升序排序（可以粗略认为这是一个单调递增地赋给每个 DDL 任务的版本号）。这些记录在 Syncer 中会用于[创建](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/syncer.go#L78)一个 Schema 对象。在运行过程中，Drainer 每遇到一条 DDL 也会[添加到 Schema 中](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/syncer.go#L367)。

binlog 中带有一个 `SchemaVersion` 信息，记录这条 binlog 生成的时刻 Schema 版本。在同步 Binlog 前，我们会先用这个 `SchemaVersion` 信息调用 Schema 的一个方法 [handlePreviousDDLJobIfNeed](https://github.com/pingcap/tidb-binlog/blob/v3.0.0/drainer/schema.go#L231)。上一段中我们看到 Schema 从何处收集到有序的 DDL Job 记录，这个方法则是按顺序应用 `SchemaVersion` 小于等于指定版本的 DDL Job，在 Schema 中维护每个表对应版本的最新结构信息，去掉一些错误代码后实现大致如下：

```
func (s *Schema) handlePreviousDDLJobIfNeed(version int64) error {
  var i int
  for i = 0; i < len(s.jobs); i++ {
     if s.jobs[i].BinlogInfo.SchemaVersion <= version {
        _, _, _, err := s.handleDDL(s.jobs[i])
        if err != nil {
           return errors.Annotatef(err, "handle ddl job %v failed, the schema info: %s", s.jobs[i], s)
        }
     } else {
        break
     }
  }

  s.jobs = s.jobs[i:]

  return nil
}
```

对于每个符合条件的 Job，由 `handleDDL` 方法将其表结构 TableInfo 等信息更新到 `Schema` 中，其他模块就可以查询到表格当前最新的信息。

## 恢复工具

我们知道 Drainer 除了可以将 binlog 直接还原到下游数据库以外，还支持同步到其他外部存储系统块，所以我们也提供了相应的工具来处理存储下来的文件，`Reparo` 是其中之一，用于读取存储在文件系统中的 binlog 文件，写入 TiDB 中。本节简单介绍下 Reparo 的用途与实现，读者可以作为示例了解如何处理同步到文件系统的 binlog 增量备份。

### Reparo

[Reparo](https://github.com/pingcap/tidb-binlog/tree/v3.0.0/reparo) 可以读取同步到文件系统上的 binlog 增量备份并同步到 TiDB。

#### 读取 binlog

当下游设置成 File（增量备份） 时，Drainer 会将 Protobuf 编码的 binlog 保存到指定目录，每写满 512 MB 新建一个文件。每个文件有个编号，从 0 开始依次类推。文件名格式定义如下：

```
// BinlogName creates a binlog file name. The file name format is like binlog-0000000000000001-20181010101010
func BinlogName(index uint64) string {
  currentTime := time.Now()
  return binlogNameWithDateTime(index, currentTime)
}

// binlogNameWithDateTime creates a binlog file name.
func binlogNameWithDateTime(index uint64, datetime time.Time) string {
  return fmt.Sprintf("binlog-%016d-%s", index, datetime.Format(datetimeFormat))
}
```

文件的前缀都是 “binlog-”，后面跟一个 16 位右对齐的编号和一个时间戳。将目录里的文件按字母顺序排序就可以得到按编号排序的 binlog 文件名。从指定目录获取文件列表的实现如下：

```
// ReadDir reads and returns all file and dir names from directory
func ReadDir(dirpath string) ([]string, error) {
  dir, err := os.Open(dirpath)
  if err != nil {
     return nil, errors.Trace(err)
  }
  defer dir.Close()

  names, err := dir.Readdirnames(-1)
  if err != nil {
     return nil, errors.Annotatef(err, "dir %s", dirpath)
  }

  sort.Strings(names)

  return names, nil
}
```

这个函数简单地获取目录里全部文件名，排序后返回。在上层还做了一些过滤来去掉临时文件等。得到文件列表后，`Reparo` 会用标准库的 [bufio.NewReader](https://golang.org/pkg/bufio/#NewReader) 逐个打开文件，然后用 `Decode` 函数读出其中的一条条 binlog：

```
func Decode(r io.Reader) (*pb.Binlog, int64, error) {
  payload, length, err := binlogfile.Decode(r)
  if err != nil {
     return nil, 0, errors.Trace(err)
  }

  binlog := &pb.Binlog{}
  err = binlog.Unmarshal(payload)
  if err != nil {
     return nil, 0, errors.Trace(err)
  }
  return binlog, length, nil
}
```

这里先调用了 `binlogfile.Decode` 从文件中解析出对应 Protobuf 编码的一段二进制数据然后解码出 binlog。

#### 写入 TiDB

得到 binlog 后就可以准备写入 TiDB。Reparo 这部分实现像一个简化版的 Drainer 的 `Sync` 模块，同样有一个 Syncer 接口以及几个具体实现（除了 `mysqlSyncer` 还有用于调试的 `printSyncer` 和 `memSyncer`），所以就不再介绍。值得一提的是，这里也跟前面很多 MySQL / TiDB 同步相关的模块一样使用了 loader 模块。

## 小结

本文介绍了 Drainer 是如何实现数据同步的以及 Reparo 如何从文件系统中恢复增量备份数据到 MySQL / TiDB。在 Drainer 中，Syncer 封装了同步到各个下游模块的具体细节，Checkpoint 记录同步进度，Translator 从 binlog 中还原出具体的变更，Schema 在内存中维护每个表对应的表结构定义。

TiDB Binlog 源码阅读系列在此就全部完结了，相信大家通过本系列文章更全面地理解了 TiDB Binlog 的原理和实现细节。我们将继续打磨优化，欢迎大家给我们反馈使用过程中遇到的问题或建议；如果社区小伙伴们想参与 TiDB Binlog 的设计、开发和测试，也欢迎与我们联系 [info@pingcap.com](mailto:info@pingcap.com)，或者在 Repo 中[提 issue](https://github.com/pingcap/tidb-binlog/issues) 讨论。
