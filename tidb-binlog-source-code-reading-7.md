---
title: TiDB Binlog 源码阅读系列文章（七）Drainer server 介绍
author: ['黄佳豪']
date: 2019-12-24
summary: 本文介绍了 Drainer server 的实现。
tags: ['TiDB Binlog 源码阅读','社区']
---

前面文章介绍了 Pump server，接下来我们来介绍 Drainer server 的实现，Drainer server 的主要作用是从各个 Pump server 获取 binlog，按 commit timestamp 归并排序后解析 binlog 同步到不同的目标系统，对应的源码主要集中在 TiDB Binlog 仓库的 [drainer/](https://github.com/pingcap/tidb-binlog/tree/v3.0.7/drainer) 目录下。

## 启动 Drainer Server

Drainer server 的启动逻辑主要实现在两个函数中：[NewServer](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/server.go#L88) 和 [(*Server).Start()](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/server.go#L250) 。

`NewServer` 根据传入的配置项创建 Server 实例，初始化 Server 运行所需的字段。其中重要字段的说明如下：

1.  metrics: [MetricClient](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/pkg/util/p8s.go#L36)，用于定时向 Prometheus Pushgateway 推送 drainer 运行中的各项参数指标。

2.  cp: [checkpoint](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/checkpoint/checkpoint.go#L29)，用于保存 drainer 已经成功输出到目标系统的 binlog 的 commit timestamp。drainer 在重启时会从 checkpoint 记录的 commit timestamp 开始同步 binlog。

3.  collector: [collector](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/collector.go#L50)，用于收集全部 binlog 数据并按照 commit timestamp 递增的顺序进行排序。同时 collector 也负责实时维护 pump 集群的状态信息。

4.  syncer: [syncer](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/syncer.go#L39)，用于将排好序的 binlog 输出到目标系统 (MySQL，Kafka...) ，同时更新同步成功的 binlog 的 commit timestamp 到 checkpoint。

Server 初始化以后，就可以用 `(*Server).Start` 启动服务，启动的逻辑包含：

1.  初始化 `heartbeat` 协程定时上报心跳信息到 etcd （内嵌在 PD 中）。

2.  调用 `collector.Start()` 驱动 `Collector` 处理单元。

3.  调用 `syncer.Start()` 驱动 `Syncer` 处理单元。

    ```go
    errc := s.heartbeat(s.ctx)
    go func() {
        for err := range errc {
            log.Error("send heart failed", zap.Error(err))
        }
    }()

    s.tg.GoNoPanic("collect", func() {
        defer func() { go s.Close() }()
        s.collector.Start(s.ctx)
    })

    if s.metrics != nil {
        s.tg.GoNoPanic("metrics", func() {
    ```

后续的章节中，我们会详细介绍 Checkpoint、Collector 与 Syncer。

## Checkpoint

Checkpoint 代码在 [/drainer/checkpoint](https://github.com/pingcap/tidb-binlog/tree/v3.0.7/drainer/checkpoint) 下。

首先看下 [接口定义](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/checkpoint/checkpoint.go#L29)：

```go
// When syncer restarts, we should reload meta info to guarantee continuous transmission.
type CheckPoint interface {
    // Load loads checkpoint information.
    Load() error

    // Save saves checkpoint information.
    Save(int64, int64) error

    // TS get the saved commit ts.
    TS() int64

    // Close closes the CheckPoint and release resources after closed other methods should not be called again.
    Close() error
}
```

drainer 支持把 checkpoint 保存到不同类型的存储介质中，目前支持 mysql 和 file 两种类型，例如 mysql 类型的实现代码在 [mysql.go](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/checkpoint/mysql.go) 。如果用户没有指定 checkpoit 的存储类型，drainer 会根据目标系统的类型自动选择对应的 checkpoint 存储类型。

当目标系统是 mysql/tidb，drainer 默认会保存 checkpoint 到 `tidb_binlog.checkpoint` 表中：

```shell
mysql> select * from tidb_binlog.checkpoint;
+---------------------+---------------------------------------------+
| clusterID           | checkPoint                                  |
+---------------------+---------------------------------------------+
| 6766844929645682862 | {"commitTS":413015447777050625,"ts-map":{}} |
+---------------------+---------------------------------------------+
1 row in set (0.00 sec)
```

commitTS 表示这个时间戳之前的数据都已经同步到目标系统了。ts-map 是用来做 [TiDB 主从集群的数据校验](https://pingcap.com/docs-cn/stable/reference/tools/sync-diff-inspector/tidb-diff/) 而保存的上下游 snapshot 对应关系的时间戳。

下面看看 MysqlCheckpoint 主要方法的实现。

```go
// Load implements CheckPoint.Load interface
func (sp *MysqlCheckPoint) Load() error {
    sp.Lock()
    defer sp.Unlock()

    if sp.closed {
        return errors.Trace(ErrCheckPointClosed)
    }

    defer func() {
        if sp.CommitTS == 0 {
            sp.CommitTS = sp.initialCommitTS
        }
    }()

    var str string
    selectSQL := genSelectSQL(sp)
    err := sp.db.QueryRow(selectSQL).Scan(&str)
    switch {
    case err == sql.ErrNoRows:
        sp.CommitTS = sp.initialCommitTS
        return nil
    case err != nil:
        return errors.Annotatef(err, "QueryRow failed, sql: %s", selectSQL)
    }

    if err := json.Unmarshal([]byte(str), sp); err != nil {
        return errors.Trace(err)
    }

    return nil
}
```

Load 方法从数据库中读取 checkpoint 信息。需要注意的是，如果 drainer 读取不到对应的 checkpoint，会使用 drainer 配置的 `initial-commit-ts` 做为启动的开始同步点。

```go
// Save implements checkpoint.Save interface
func (sp *MysqlCheckPoint) Save(ts, slaveTS int64) error {
    sp.Lock()
    defer sp.Unlock()

    if sp.closed {
        return errors.Trace(ErrCheckPointClosed)
    }

    sp.CommitTS = ts

    if slaveTS > 0 {
        sp.TsMap["master-ts"] = ts
        sp.TsMap["slave-ts"] = slaveTS
    }

    b, err := json.Marshal(sp)
    if err != nil {
        return errors.Annotate(err, "json marshal failed")
    }

    sql := genReplaceSQL(sp, string(b))
    _, err = sp.db.Exec(sql)
    if err != nil {
        return errors.Annotatef(err, "query sql failed: %s", sql)
    }

    return nil
}
```

Save 方法构造对应 SQL 将 checkpoint 写入到目标数据库中。

## Collector

Collector 负责获取全部 binlog 信息后，按序传给 Syncer 处理单元。我们先看下 Start 方法：

```go
// Start run a loop of collecting binlog from pumps online
func (c *Collector) Start(ctx context.Context) {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        c.publishBinlogs(ctx)
        wg.Done()
    }()

    c.keepUpdatingStatus(ctx, c.updateStatus)

    for _, p := range c.pumps {
        p.Close()
    }
    if err := c.reg.Close(); err != nil {
        log.Error(err.Error())
    }
    c.merger.Close()

    wg.Wait()
}
```

这里只需要关注 publishBinlogs 和 keepUpdatingStatus 两个方法。

```go
func (c *Collector) publishBinlogs(ctx context.Context) {
    defer log.Info("publishBinlogs quit")

    for {
        select {
        case <-ctx.Done():
            return
        case mergeItem, ok := <-c.merger.Output():
            if !ok {
                return
            }
            item := mergeItem.(*binlogItem)
            if err := c.syncBinlog(item); err != nil {
                c.reportErr(ctx, err)
                return
            }
        }
    }
}
```

publishBinlogs 调用 [merger](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/merge.go) 模块从所有 pump 读取 binlog，并且按照 binlog 的 commit timestamp 进行归并排序，最后通过调用 `syncBinlog` 输出 binlog 到  Syncer 处理单元。

```go
func (c *Collector) keepUpdatingStatus(ctx context.Context, fUpdate func(context.Context) error) {
    // add all the pump to merger
    c.merger.Stop()
    fUpdate(ctx)
    c.merger.Continue()

    // update status when had pump notify or reach wait time
    for {
        select {
        case <-ctx.Done():
            return
        case nr := <-c.notifyChan:
            nr.err = fUpdate(ctx)
            nr.wg.Done()
        case <-time.After(c.interval):
            if err := fUpdate(ctx); err != nil {
                log.Error("Failed to update collector status", zap.Error(err))
            }
        case err := <-c.errCh:
            log.Error("collector meets error", zap.Error(err))
            return
        }
    }
}
```

keepUpdatingStatus 通过下面两种方式从 etcd 获取 pump 集群的最新状态：

1.  定时器定时触发。

2.  notifyChan 触发。这是一个必须要提一下的处理逻辑：当一个 pump 需要加入 pump c 集群的时候，该 pump 会在启动时通知所有在线的 drainer，只有全部 drainer 都被通知都成功后，pump 方可对外提供服务。 这个设计的目的是，防止对应的 pump 的 binlog 数据没有及时加入 drainer 的排序过程，从而导致 binlog 数据同步缺失。

## Syncer

Syncer 代码位于 [drainer/syncer.go](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/syncer.go)，是用来处理数据同步的关键模块。

```go
type Syncer struct {
    schema *Schema
    cp     checkpoint.CheckPoint
    cfg    *SyncerConfig
    input  chan *binlogItem
    filter *filter.Filter
    // last time we successfully sync binlog item to downstream
    lastSyncTime time.Time
    dsyncer      dsync.Syncer
    shutdown     chan struct{}
    closed       chan struct{}
}
```

在 Syncer 的结构定义中，我们关注下面三个对象：

*   dsyncer 是真正同步数据到不同目标系统的执行器实现，我们会在后续章节具体介绍，接口定义如下：

    ```go
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

*   schema 维护了当前同步位置点的全部 schema 信息，可以根据 ddl binlog 变更对应的 schema 信息。

*   filter 负责对需要同步的 binlog 进行过滤。

Syncer 运行入口在 [run](https://github.com/pingcap/tidb-binlog/blob/v3.0.7/drainer/syncer.go#L260) 方法，主要逻辑包含：

1.  依次处理 Collector 处理单元推送过来的 binlog 数据。

2.  如果是 DDL binlog，则更新维护的 schema 信息。

3.  利用 filter 过滤不需要同步到下游的数据。

4.  调用 drainer/sync/Syncer.Sync()  异步地将数据同步到目标系统。

5.  处理数据同步结果返回。

    a. 通过 Succsses() 感知已经成功同步到下游的 binlog 数据，保存其对应 commit timestamp 信息到 checkpoint。
  
    b. 通过 Error() 感知同步过程出现的错误，drainer 清理环境退出进程。

## 小结

本文介绍了 Drainer server 的主体结构，后续文章会具体介绍其如何同步数据到不同下游。