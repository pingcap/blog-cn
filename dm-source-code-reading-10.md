---
title: DM 源码阅读系列文章（十）测试框架的实现
author: ['杨非']
date: 2019-07-23
summary: 本篇文章将从质量保证的角度来介绍 DM 测试框架的设计和实现，探讨如何通过多维度的的测试方法保证 DM 的正确性和稳定性。 
tags: ['DM 源码阅读','社区']
---

本文为 TiDB Data Migration 源码阅读系列文章的第十篇，之前的文章已经详细介绍过 DM 数据同步各组件的实现原理和代码解析，相信大家对 DM 的实现细节已经有了深入的了解。本篇文章将从质量保证的角度来介绍 DM 测试框架的设计和实现，探讨如何通过多维度的测试方法保证 DM 的正确性和稳定性。

## 测试体系

DM 完整的测试体系包括以下四个部分：

### 1. 单元测试

主要用于测试每个 go 模块和具体函数实现的正确性，测试用例编写和测试运行方式依照 go 单元测试的标准，测试代码跟随项目源代码一起发布。具体测试用例编写使用 [pingcap/check](https://github.com/pingcap/check) 工具包，该工具包是在 go 原生测试工具基础上进行的扩展，[按照 suite 分组进行测试](https://github.com/pingcap/check/blob/67f458068fc864dabf17e38d4d337f28430d13ed/run.go#L98-L131)，提供包括更丰富的检测语法糖、并行测试、序列化测试在内的一些扩展特性。单元测试的设计出发点是白盒测试，测试用例中通过尽可能明确的测试输入得到期望的测试输出。

### 2. 集成测试

用于测试各个组件之间交互的正确性和完整数据同步流程的正确性，完整的 [测试用例集合和测试工具在项目代码的 tests 目录](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests) 发布。集成测试首先自定义了一些 [DM 基础测试工具集](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/_utils)，包括启动 DM 组件，生成、导入测试数据，检测同步状态、上下游数据一致性等 bash 脚本，每个测试用例是一个完整的数据同步场景，通过脚本实现数据准备、启动 DM 集群、模拟上游数据输入、特定异常和恢复、数据同步校验等测试流程。集成测试的设计出发点是确定性的模拟测试场景，为了能够确定性的模拟一些特定的同步场景，为此我们还引入了 [failpoint](https://github.com/pingcap/failpoint) 来注入测试、控制测试流程， 以及 [trace](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/pkg/tracing) 机制来更准确地获取程序内存状态、辅助控制测试流程，具体的实现细节会在后文详细介绍。

### 3. 破坏性测试

真实的软件运行环境中会遇到各种各样的问题，包括各类硬件故障、网络延迟和隔离、资源不足等等。DM 在数据同步过程中也同样会遇到这些问题，借助于 [PingCAP 内部的自动化混沌测试平台 schrodinger](https://thenewstack.io/chaos-tools-and-techniques-for-testing-the-tidb-distributed-newsql-database/)，我们设计了多个破坏性测试用例，包括在同步过程中随机 kill DM-worker 节点，同步过程中重启部分 DM-worker 节点，分发不兼容 DDL 语句等测试场景。这一类测试的关注点是在各类破坏性操作之后数据同步能否正常恢复以及验证在这些场景下数据一致性的保证，测试用例通常以黑盒的形式去运行，并且长期、反复地进行测试。

### 4. 稳定性测试

目前该类测试运行在 PingCAP 内部的 K8s 集群上，通常每个测试的应用规模会比较大，譬如有一些 100+ 上游实例，300+ 分库分表合并的测试场景，数据负载也会相对较高，目标在于测试大规模 DM 集群在高负载下长期运行的稳定性。该类测试也属于黑盒测试，每个测试用例内会根据任务配置启动上游的 MySQL 集群、DM 集群、下游 TiDB 集群和数据导入集群。上游数据输入工具有多种，包括 [随机 DML 生成工具](https://github.com/amyangfei/data-dam)，schrodinger 测试用例集等。具体的测试 case 和 K8s 部署脚本可以在 [dm-K8s 仓库](https://github.com/csuzhangxc/dm-k8s) 找到。

### 5. 测试方法对比

我们通过以下的表格对比不同测试维度在测试体系中发挥的作用和它们之间的互补性。

| 测试名称 | 测试方法 | 测试重点 | 测试周期 | 测试互补性 |
|:-------------:|:-----------|:-----------|:--------------|:----------|
| 单元测试 | 白盒测试，确定性的输入、输出 | 模块和具体函数的正确性 | CI 自动化触发，新代码提交前必须通过 | 保证单个函数的正确性 |
| 集成测试 | 确定性的同步场景和数据负载 | 模块之间整体交互的正确性，可以有针对性的测试特定数据同步场景。 | CI 自动化触发，新代码提交前必须通过测试 | 在单元测试的基础上，保证多个模块在一起组合起来工作的正确性 |
| 破坏性测试 | 黑盒测试，随机数据，随机触发的固定类型外部扰动 | 系统在异常场景下的稳定性和正确性 | 在内部测试平台长期、反复运行 | 对已有确定输入测试的补充，增加测试输入的不确定性，通过未知、随机的外部扰动发现系统潜在的问题 |
| 长期稳定性测试 | 黑盒测试，确定性的同步场景，随机数据负载 | 系统长期运行的稳定性和正确性 | 在内部 K8s 集群长期运行 | 补充集成测试的场景，测试系统在更高负载、更长运行时间内的表现 |

## 测试 case 与测试工具的实现 

### 1. 在单元测试中进行 mock

我们在单元测试运行过程中希望尽量减少外部环境或内部组件的依赖，譬如测试 relay 模块时我们并不希望从上游的 MySQL 拉取 binlog，或者测试到下游的一些数据库读写操作并不希望真正部署一个下游 TiDB，这时候我们就需要对测试 case 进行适当的 mock。在单元测试中针对不同的场景采用了多种 mock 方案。接下来我们选取几种具有代表性的方案进行介绍。

#### Mock golang interface

在 golang 中只要调用者本身实现了接口的全部方法，就默认实现了该接口，这一特性使得使用接口方法调用的代码具有良好的扩展性，对于测试也提供了天然的 mock 方法。以 worker 内部各 subtask 的 [任务暂停、恢复的测试用例](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/worker/subtask_test.go#L258) 为例，测试过程中会涉及到 dump unit 和 load unit 的运行、出错、暂停和恢复等操作。我们定义 [MockUnit](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/worker/subtask_test.go#L67-L76) 并且实现了 [unit interface](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/unit/unit.go#L24) 的 [全部方法](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/worker/subtask_test.go#L86-L124)，就可以在单元测试里模拟任务中 unit 的各类操作。还可以定义 [各类注入函数](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/worker/subtask_test.go#L126-L143)，实现控制某些逻辑流程中的出错测试和执行路径控制。

#### 自定义 binlog 生成工具

在前文已经介绍过 [relay 处理单元从上游读取 binlog 并写入本地文件](https://pingcap.com/blog-cn/dm-source-code-reading-6/) 的实现细节，这一过程重度依赖于 MySQL binlog 的处理和解析。为了在单元测试中完善模拟 binlog 数据流，DM 中实现了一个 [binlog 生成工具](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/pkg/binlog/event)，该工具包提供了通用的 [generator](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/pkg/binlog/event/generator.go#L25) 用于连续生成 Event 以及相对底层的生成特定 Event 的接口，支持 MySQL 和 MariaDB 两种数据库的 binlog 协议。generator 提供的生成接口会返回一个 go-mysql 的 [`BinlogEvent`](https://github.com/siddontang/go-mysql/blob/7ed1210c02a2867a8d4570f526422af9fcd4246b/replication/event.go#L25) 列表和 binlog 对应的 byte 数组，同时在 generator 中自动更新 binlog 位置信息和 `GTID` 信息。类似的，更底层的生成 Event 接口会要求提供数据类型、`serverID`、`latestPos`、`latestGTID` 以及可能需要的库名、表名、SQL 语句等信息，生成的结果是一个 [`DDLDMLResult`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/pkg/binlog/event/common.go#L28) 对象。

我们通过测试中的一个 case 来了解如何使用这个工具，以 [relay 模块读取到多个 binlog event 写入文件的正确性测试](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L370) 这个 case 为例：

1. [首先配置数据库类型，`serverID`，`GTID` 和 `XID` 相关信息，初始化 relay log 写入目录和文件名](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L371-L387)

2. [初始化 `allEvents` 数组](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L390)，用于模拟从上游接收到的 `replication.BinlogEvent`；[初始化 `allData`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L391)，`allData` 存储 binlog binary 数据，用于后续 relay log 写入的验证；[初始化 `generator`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L392)

3. [通过 generator `GenFileHeader` 接口生成 `replication.BinlogEvent` 和 binlog 数据](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L396)（对应的 binlog 中包含 `FormatDescriptionEvent` 和 `PreviousGTIDsEvent`）。生成的 [`replication.BinlogEvent` 保存到 `allEvents`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L398)，[binlog 数据保存到 `allData`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L399)。

4. 按照 3 的操作流程分别[生成 `CREATE DATABASE`，`CREATE TABLE` 和一条 `INSERT` 语句对应的 event/binlog 数据并保存](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L402-L421)

5. [创建 `relay.FileWriter`，按照顺序读取 3, 4 步骤中保存的 `replication.BinlogEvent`，向配置的 relay log 文件中写入 relay log](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L424-L430)

6. [检查 relay log 文件写入的数据长度与 `allData` 存储的数据长度相同](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L432)

7.  [读取 relay log 文件，检查数据内容和 `allData` 存储的数据内容相同](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/relay/writer/file_test.go#L435-L438)

至此我们就结合 binlog 生成工具完成了一个 relay 模块的测试 case。目前 DM 已经在很多 case 中使用 binlog 生成工具模拟生成 binlog，仍然存在的 [少量 case](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/syncer/syncer_test.go) 依赖上游数据库生成 binlog，我们已经计划借助 binlog 生成工具移除这些外部依赖。

#### 其他 mock 工具

* 在验证数据库读写操作逻辑正确性的测试中，使用了 [go-sqlmock](https://github.com/DATA-DOG/go-sqlmock) 来 mock sql driver 的行为。

* 在验证 gRPC 交互逻辑的正确性测试中，使用了 [官方提供的 mock 工具](https://github.com/golang/mock)，针对 gRPC 接口生成 mock 文件，在此基础上测试 gRPC 接口和应用逻辑的正确性。

### 2. 集成测试的方法和相关工具

#### Trace 信息收集

DM 内部定义了一个简单的信息 trace 收集工具，其设计目标是在 DM 运行过程中，通过增加代码内部的埋点，定期收集系统运行时的各类信息。trace 工具包含一个提供 gRPC 上报信息接口和 HTTP 控制接口的 [tracer 服务器](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/tracer) 和提供埋点以及后台收集信息上传功能的 [tracing 包](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/pkg/tracing)。tracing 模块上传到 tracer 服务器的事件数据通过 `protobuf` 进行定义，[`BaseEvent`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/proto/tracer_base.proto#L11-L18) 定义了最基本的 trace 事件，包含了运行代码文件名、代码行、事件时间戳、事件 ID、事件组 ID 和事件类型，用户自定义的事件需要包含 `BaseEvent`。tracing 模块会 [定期向 tracer 服务器同步全局时间戳](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/pkg/tracing/tracer.go#L129)，通过这种方式保证多节点不同的 trace 事件会保持大致的时间顺序（注意这里并不是严格的时间序，会依赖于每分钟内本地时钟的准确性，仍然有各种出现乱序的可能）。设计 tracing 模块的主要目的有以下两点：

*  对于同一个 DM 组件（DM-master/DM-worker），希望记录一些重要内存信息的数据流历史。例如在 binlog replication 处理单元处理一条 query event 过程中会经历处理 binlog event 、生成 ddl job、执行 job 这三个阶段，我们将这三个处理逻辑抽象为三个事件，三个事件在时间上是有先后关系的，在逻辑上关联了同一个 binlog 的处理流程，在 DM 中记录这三个事件的 trace event 时使用了同一个 `traceID`（[处理 binlog event 生成一个新的 traceID](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/syncer/syncer.go#L1597)，该 `traceID` 记录在 ddl job 中，[分发 ddl job 时记录的 trace 事件会复用此 traceID](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/syncer/syncer.go#L688)；[在 executor 中最后执行 ddl job 的过程中记录的 trace 事件也会复用此 `traceID`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/syncer/syncer.go#L864)），这样就将三个事件关联起来，因为在同一个进程内，他们的时间戳真实反映了时间维度上的顺序关系。

*  由于 DM 提供了 shard DDL 的机制，多个 DM-worker 之间的数据会存在关联，譬如在进行 shard DDL 的过程中，处于同一个 shard group 内的多个 DM-worker 的 DDL 是关联在一起的。`BaseEvent` 定义中的 [`groupID`](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/proto/tracer_base.proto#L16) 字段就是用来解决多进程间 trace 事件关联性的问题，定义具有相同 `groupID` 的事件属于同一个事件组，表示它们之间在逻辑上有一定关联性。举一个例子，在 shard DDL 这个场景下，DM-master 协调 shard DDL 时会分别 [向 DDL owner 分发执行 SQL 的请求](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/master/server.go#L1423-L1432)，以及 [向非 owner 分发忽略 DDL 的请求](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/dm/master/server.go#L1457-L1466)，在这两组请求中携带了相同的 `groupID`，binlog replication 分发 ddl job 时会获取到 `groupID`，这样就将不同进程间 shard DDL 的执行关联了起来。

我们可以利用收集的 trace 信息辅助验证数据同步的正确性。譬如在 [验证 `safe_mode` 逻辑正确性的测试](https://github.com/pingcap/dm/tree/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/safe_mode) 中，[我们将 DM 启动阶段的 `safe_mode` 时间调短为 0s](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/safe_mode/run.sh#L35)，期望验证对于上游 update 操作产生的 binlog，如果该操作发生时上下游 shard DDL 没有完全同步，那么同步该 binlog 时的 `safe_mode` 为 true；反之如果该操作发生时上下游没有进行 shard DDL 或 shard DDL 已经同步，那么 `safe_mode` 为 false。通过 trace 机制，可以很容易从 [tracer server 的接口获取测试过程中的所有事件信息](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/_dmctl_tools/check_safe_mode.go#L42-L55)，[并且抽取出 update DML，DDL 等对应的 trace event 信息](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/_dmctl_tools/check_safe_mode.go#L123-L133)，[进一步通过这些信息验证 `safe_mode` 在 shard DDL 同步场景下工作的正确性](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/_dmctl_tools/check_safe_mode.go#L167-L180)。

#### Failpoint 的使用

在集成测试中，为了对特定的同步流程或者特定的错误中断做确定性测试，我们开发了一个名为 [failpoint](https://github.com/pingcap/failpoint) 的项目，用来在代码中注入特定的错误。现阶段 DM 集成测试的 case 都是 [提前设定环境变量，然后启动 DM 相关进程来控制注入点的生效与否](https://github.com/pingcap/dm/blob/7cba6d21d78dd16e9ab159e9c0300efcbdeb1e4a/tests/safe_mode/run.sh#L35-L38)。目前我们正在探索将 trace 和 failpoint 结合的方案，通过 trace 获取进程内部状态，借助 failpoint 提供的 http 接口动态调整注入点，以实现更智能、更通用的错误注入测试。

### 3. 破坏性测试和大规模测试的原理与展望

#### 破坏性测试中的错误注入

目前破坏性测试的测试 case 并没有对外开源，我们在这里介绍 DM 破坏性测试中所使用的部分故障注入

* 使用 `kill -9` 强制终止 DM-worker 进程，或者使用 `kill` 来优雅地终止进程，然后重新启动

* 模拟上游写入 TiDB 不兼容的 DDL，通过 `sql-skip/sql-replace` 跳过或替换不兼容 DDL 恢复同步的场景

* 模拟上游发生主从切换时 DM 进行主从切换处理的正确性

* 模拟下游 TiDB/TiKV 故障不可写入的场景

* 模拟网络出现丢包或高延迟的场景

* 在未来 DM 提供高可用支持之后，还会增加更多的高可用相关测试场景，譬如磁盘空间写满、DM-worker 节点宕机自动恢复等

#### 大规模测试

大规模测试中的上游负载复用了很多在 TiDB 中的测试用例，譬如银行转账、大规模 DDL 操作等测试场景。该测试所有 case 均运行在 K8s 中，基于 K8s deployment yaml 部署一系列的 statefuset，通过 `configmap` 传递拓扑信息。目前 DM 正在规划实现 DM-operator 以及运行于 K8s 之上的完整解决方案，预期在未来可以更便捷地部署在 K8s 环境上，后续的大规模测试也会基于此继续展开。

## 总结

本篇文章详细地介绍了 DM 的测试体系，测试中使用到的工具和一些 case 的实例分析，分析如何通过多维度的测试保证 DM 的正确性、稳定性。然而尽管已经有了如此多的测试，我们仍不能保证 bug free，也不能保证测试 case 对于各类场景和逻辑路径进行了百分之百的覆盖，对于测试方法和测试 case 的完善仍需要不断的探索。

**至此 DM 的源码阅读系列就暂时告一段落了，但是 DM 还在不断地发展演化，DM 中长期的规划中有很多激动人心的改动和优化，譬如高可用方案的落地、DM on K8s、实时数据校验、更易用的数据迁移平台等（未来对于 DM 的一些新特性可能会有番外篇）。希望感兴趣的小伙伴可以持续关注 DM 的发展，也欢迎大家提供改进的建议和提 [PR](https://github.com/pingcap/dm/pulls)。**
