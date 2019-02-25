---
title: TiDB 源码阅读系列文章（二十三）Prepare/Execute 请求处理
author: ['苏立']
date: 2019-01-03
summary: 在《（三）SQL 的一生》中，我们介绍了 TiDB 在收到客户端请求包时，最常见的 `Command --- COM_QUERY` 的请求处理流程。本文我们将介绍另外一种大家经常使用的 `Command --- Prepare/Execute` 请求在 TiDB 中的处理过程。
tags: ['TiDB 源码阅读','社区']
---

在之前的一篇文章[《TiDB 源码阅读系列文章（三）SQL 的一生》](https://pingcap.com/blog-cn/tidb-source-code-reading-3/)中，我们介绍了 TiDB 在收到客户端请求包时，最常见的 `Command --- COM_QUERY` 的请求处理流程。本文我们将介绍另外一种大家经常使用的 `Command --- Prepare/Execute` 请求在 TiDB 中的处理过程。

## Prepare/Execute Statement 简介

首先我们先简单回顾下客户端使用 Prepare 请求过程：	

1. 客户端发起 Prepare 命令将带 “?” 参数占位符的 SQL 语句发送到数据库，成功后返回 `stmtID`。

2. 具体执行 SQL 时，客户端使用之前返回的 `stmtID`，并带上请求参数发起 Execute 命令来执行 SQL。

3. 不再需要 Prepare 的语句时，关闭 `stmtID` 对应的 Prepare 语句。

相比普通请求，Prepare 带来的好处是：

* 减少每次执行经过 Parser 带来的负担，因为很多场景，线上运行的 SQL 多是相同的内容，仅是参数部分不同，通过 Prepare 可以通过首次准备好带占位符的 SQL，后续只需要填充参数执行就好，可以做到“一次 Parse，多次使用”。

* 在开启 PreparePlanCache 后可以达到“一次优化，多次使用”，不用进行重复的逻辑和物理优化过程。

* 更少的网络传输，因为多次执行只用传输参数部分，并且返回结果 Binary 协议。

* 因为是在执行的同时填充参数，可以防止 SQL 注入风险。

* 某些特性比如 serverSideCursor 需要是通过 Prepare statement 才能使用。

TiDB 和 [MySQL 协议](https://dev.mysql.com/doc/refman/5.7/en/sql-syntax-prepared-statements.html) 一样，对于发起 Prepare/Execute 这种使用访问模式提供两种方式：

* Binary 协议：即上述的使用 `COM_STMT_PREPARE`，`COM_STMT_EXECUTE`，`COM_STMT_CLOSE` 命令并且通过 Binary 协议获取返回结果，这是目前各种应用开发常使用的方式。

* 文本协议：使用 `COM_QUERY`，并且用 `PREPARE`，`EXECUTE`，`DEALLOCATE PREPARE` 使用文本协议获取结果，这个效率不如上一种，多用于非程序调用场景，比如在 MySQL 客户端中手工执行。

下面我们主要以 Binary 协议来看下 TiDB 的处理过程。文本协议的处理与 Binary 协议处理过程比较类似，我们会在后面简要介绍一下它们的差异点。

## `COM_STMT_PREPARE`

首先，客户端发起 `COM_STMT_PREPARE`，在 TiDB 收到后会进入 [`clientConn#handleStmtPrepare`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L51)，这个函数会通过调用 [`TiDBContext#Prepare`](https://github.com/lysu/tidb/blob/source-read-prepare/server/driver_tidb.go#L305) 来进行实际 Prepare 操作并返回 [结果](https://dev.mysql.com/doc/internals/en/com-stmt-prepare-response.html) 给客户端，实际的 Prepare 处理主要在 [`session#PrepareStmt`](https://github.com/lysu/tidb/blob/source-read-prepare/session/session.go#L924) 和 [`PrepareExec`](https://github.com/lysu/tidb/blob/source-read-prepare/executor/prepared.go#L73) 中完成：

1. 调用 Parser 完成文本到 AST 的转换，这部分可以参考[《TiDB 源码阅读系列文章（五）TiDB SQL Parser 的实现》](https://pingcap.com/blog-cn/tidb-source-code-reading-5/)。

2. 使用名为 [`paramMarkerExtractor`](https://github.com/lysu/tidb/blob/source-read-prepare/executor/prepared.go#L57) 的 visitor 从 AST 中提取 “?” 表达式，并根据出现位置（offset）构建排序 Slice，后面我们会看到在 Execute 时会通过这个 Slice 值来快速定位并替换 “?” 占位符。

3. 检查参数个数是否超过 Uint16 最大值（这个是 [协议限制](https://dev.mysql.com/doc/internals/en/com-stmt-prepare-response.html)，对于参数只提供 2 个 Byte）。

4. 进行 Preprocess， 并且创建 LogicPlan， 这部分实现可以参考之前关于 [逻辑优化的介绍](https://pingcap.com/blog-cn/tidb-source-code-reading-7/)，这里生成 LogicPlan 主要为了获取并检查组成 Prepare 响应中需要的列信息。

5. 生成 `stmtID`，生成的方式是当前会话中的递增 int。

6. 保存 `stmtID` 到 `ast.Prepared` (由 AST，参数类型信息，schema 版本，是否使用 `PreparedPlanCache` 标记组成) 的映射信息到 [`SessionVars#PreparedStmts`](https://github.com/lysu/tidb/blob/source-read-prepare/sessionctx/variable/session.go#L185) 中供 Execute 部分使用。

7. 保存 `stmtID` 到 [`TiDBStatement`](https://github.com/lysu/tidb/blob/source-read-prepare/server/driver_tidb.go#L57) （由 `stmtID`，参数个数，SQL 返回列类型信息，`sendLongData` 预 `BoundParams` 组成）的映射信息保存到 [`TiDBContext#stmts`](https://github.com/lysu/tidb/blob/source-read-prepare/server/driver_tidb.go#L53)。

在处理完成之后客户端会收到并持有 `stmtID` 和参数类型信息，返回列类型信息，后续即可通过 `stmtID` 进行执行时，server 可以通过 6、7 步保存映射找到已经 Prepare 的信息。

## `COM_STMT_EXECUTE`

Prepare 成功之后，客户端会通过 `COM_STMT_EXECUTE` 命令请求执行，TiDB 会进入 [`clientConn#handleStmtExecute`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L108)，首先会通过 stmtID 在上节介绍中保存的 [`TiDBContext#stmts`](https://github.com/lysu/tidb/blob/source-read-prepare/server/driver_tidb.go#L53) 中获取前面保存的 `TiDBStatement`，并解析出是否使用 `userCursor` 和请求参数信息，并且调用对应 `TiDBStatement` 的 Execute 进行实际的 Execute 逻辑：

1. 生成 [`ast.ExecuteStmt`](https://github.com/pingcap/parser/blob/732efe993f70da99fdc18acb380737be33f2333a/ast/misc.go#L218) 并调用 [`planer.Optimize`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/optimize.go#L28) 生成 `plancore.Execute`，和普通优化过程不同的是会执行 [`Exeucte#OptimizePreparedPlan`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/optimize.go#L53)。

2. 使用 `stmtID` 通过 [`SessionVars#PreparedStmts`](https://github.com/lysu/tidb/blob/source-read-prepare/sessionctx/variable/session.go#L190) 获取到到 Prepare 阶段的 `ast.Prepared` 信息。

3. 使用上一节第 2 步中准备的 [`prepared.Params`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/core/common_plans.go#L167) 来快速查找并填充参数值；同时会保存一份参数到 [`sessionVars.PreparedParams`](https://github.com/lysu/tidb/blob/source-read-prepare/sessionctx/variable/session.go#L190) 中，这个主要用于支持 `PreparePlanCache` 延迟获取参数。

4.  判断对比判断 Prepare 和 Execute 之间 schema 是否有变化，如果有变化则重新 Preprocess。

5.  之后调用 [`Execute#getPhysicalPlan`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/core/common_plans.go#L188) 获取物理计划，实现中首先会根据是否启用 PreparedPlanCache 来查找已缓存的 Plan，本文后面我们也会专门介绍这个。

6.  在没有开启 PreparedPlanCache 或者开启了但没命中 cache 时，会对 AST 进行一次正常的 Optimize。

在获取到 PhysicalPlan 后就是正常的 [Executing 执行](https://zhuanlan.zhihu.com/p/35134962)。

## `COM_STMT_CLOSE`

在客户不再需要执行之前的 Prepared 的语句时，可以通过 `COM_STMT_CLOSE` 来释放服务器资源，TiDB 收到后会进入 [`clientConn#handleStmtClose`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L501)，会通过 `stmtID` 在 `TiDBContext#stmts` 中找到对应的 `TiDBStatement`，并且执行 [Close](https://github.com/lysu/tidb/blob/source-read-prepare/server/driver_tidb.go#L152) 清理之前的保存的 `TiDBContext#stmts` 和 `SessionVars#PrepareStmts`，不过通过代码我们看到，对于前者的确直接进行了清理，对于后者不会删除而是加入到 [`RetryInfo#DroppedPreparedStmtIDs`](https://github.com/lysu/tidb/blob/source-read-prepare/session/session.go#L1020) 中，等待当前事务提交或回滚才会从 `SessionVars#PrepareStmts` 中清理，之所以延迟删除是由于 TiDB 在事务提交阶段遇到冲突会根据配置决定是否重试事务，参与重试的语句可能只有 Execute 和 Deallocate，为了保证重试还能通过 `stmtID` 找到 prepared 的语句 TiDB 目前使用延迟到事务执行完成后才做清理。

## 其他 `COM_STMT`

除了上面介绍的 3 个 `COM_STMT`，还有另外几个 `COM_STMT_SEND_LONG_DATA`，`COM_STMT_FETCH`，`COM_STMT_RESET` 也会在 Prepare 中使用到。

### `COM_STMT_SEND_LONG_DATA`

某些场景我们 SQL 中的参数是 `TEXT`，`TINYTEXT`，`MEDIUMTEXT`，`LONGTEXT` and `BLOB`，`TINYBLOB`，`MEDIUMBLOB`，`LONGBLOB` 列时，客户端通常不会在一次 Execute 中带大量的参数，而是单独通过 [`COM_SEND_LONG_DATA`](https://dev.mysql.com/doc/internals/en/com-stmt-send-long-data.html) 预先发到 TiDB，最后再进行 Execute。

TiDB 的处理在 [`client#handleStmtSendLongData`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L514)，通过 `stmtID` 在 `TiDBContext#stmts` 中找到 `TiDBStatement` 并提前放置 `paramID` 对应的参数信息，进行追加参数到 `boundParams`（所以客户端其实可以多次 send 数据并追加到一个参数上），Execute 时会通过 `stmt.BoundParams()` 获取到提前传过来的参数并和 Execute 命令带的参数 [一起执行](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L176)，在每次执行完成后会重置 `boundParams`。

### `COM_STMT_FETCH`

通常的 Execute 执行后，TiDB 会向客户端持续返回结果，返回速率受 `max_chunk_size` 控制（见《[TiDB 源码阅读系列文章（十）Chunk 和执行框架简介](https://pingcap.com/blog-cn/tidb-source-code-reading-10/)》）， 但实际中返回的结果集可能非常大。客户端受限于资源（一般是内存）无法一次处理那么多数据，就希望服务端一批批返回，[`COM_STMT_FETCH`](https://dev.mysql.com/doc/internals/en/com-stmt-fetch.html) 正好解决这个问题。

它的使用首先要和 `COM_STMT_EXECUTE` 配合（也就是必须使用 Prepared 语句执行）， `handleStmtExeucte` 请求协议 flag 中有标记要使用 cursor，execute 在完成 plan 拿到结果集后并不立即执行而是把它缓存到 `TiDBStatement` 中，并立刻向客户端回包中带上列信息并标记 [`ServerStatusCursorExists`](https://dev.mysql.com/doc/internals/en/status-flags.html)，这部分逻辑可以参看 [`handleStmtExecute`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L193)。

客户端看到 `ServerStatusCursorExists` 后，会用 `COM_STMT_FETCH` 向 TiDB 拉去指定 fetchSize 大小的结果集，在 [`connClient#handleStmtFetch`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L210) 中，会通过 session 找到 `TiDBStatement` 进而找到之前缓存的结果集，开始实际调用执行器的 Next 获取满足 fetchSize 的数据并返回客户端，如果执行器一次 Next 超过了 fetchSize 会只返回 fetchSize 大小的数据并把剩下的数据留着下次再给客户端，最后对于结果集最后一次返回会标记 [`ServerStatusLastRowSend`](https://dev.mysql.com/doc/internals/en/status-flags.html) 的 flag 通知客户端没有后续数据。

### `COM_STMT_RESET`

主要用于客户端主动重置 `COM_SEND_LONG_DATA` 发来的数据，正常 `COM_STMT_EXECUTE` 后会自动重置，主要针对客户端希望主动废弃之前数据的情况，因为 `COM_STMT_SEND_LONG_DATA` 是一直追加的操作，客户端某些场景需要主动放弃之前预存的参数，这部分逻辑主要位于 [`connClient#handleStmtReset`](https://github.com/lysu/tidb/blob/source-read-prepare/server/conn_stmt.go#L531) 中。

## Prepared Plan Cache

通过前面的解析过程我们看到在 Prepare 时完成了 AST 转换，在之后的 Execute 会通过 `stmtID` 找之前的 AST 来进行 Plan 跳过每次都进行 Parse SQL 的开销。如果开启了 Prepare Plan Cache，可进一步在 Execute 处理中重用上次的 PhysicalPlan 结果，省掉查询优化过程的开销。

TiDB 可以通过 [修改配置文件](https://github.com/lysu/tidb/blob/source-read-prepare/config/config.toml.example#L167) 开启 Prepare Plan Cache， 开启后每个新 Session 创建时会初始化一个 [`SimpleLRUCache`](https://github.com/lysu/tidb/blob/source-read-prepare/util/kvcache/simple_lru.go#L38) 类型的 `preparedPlanCache` 用于保存用于缓存 Plan 结果，缓存的 key 是 `pstmtPlanCacheKey`（由当前 DB，连接 ID，`statementID`，`schemaVersion`， `snapshotTs`，`sqlMode`，`timezone` 组成，所以要命中 plan cache 这以上元素必须都和上次缓存的一致），并根据配置的缓存大小和内存大小做 LRU。

在 Execute 的处理逻辑 [`PrepareExec`](https://github.com/lysu/tidb/blob/source-read-prepare/executor/prepared.go#L161) 中除了检查 `PreparePlanCache` 是否开启外，还会判断当前的语句是否能使用 `PreparePlanCache`。

1. 只有 `SELECT`，`INSERT`，`UPDATE`，`DELETE` 有可能可以使用 `PreparedPlanCache`	。

2. 并进一步通过 [`cacheableChecker`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/core/cacheable_checker.go#L43) visitor 检查 AST 中是否有变量表达式，子查询，"order by ?"，"limit ?，?" 和 UnCacheableFunctions 的函数调用等不可以使用 PlanCache 的情况。

如果检查都通过则在 `Execute#getPhysicalPlan` 中会用当前环境构建 cache key 查找 `preparePlanCache`。

### 未命中 Cache

我们首先来看下没有命中 Cache 的情况。发现没有命中后会用 `stmtID` 找到的 AST 执行 [Optimize](https://github.com/lysu/tidb/blob/source-read-prepare/executor/prepared.go#L161)，但和正常执行 Optimize 不同对于 Cache 的 Plan， 我需要对 “?” 做延迟求值处理， 即将占位符转换为一个 function 做 Plan 并 Cache， 后续从 Cache 获取后 function 在执行时再从具体执行上下文中实际获取执行参数。

回顾下构建 LogicPlan 的过程中会通过 [`expressionRewriter`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/core/expression_rewriter.go#L151) 将 AST 转换为各类 [`expression.Expression`](https://github.com/lysu/tidb/blob/source-read-prepare/expression/expression.go#L42)，通常对于 [`ParamMarkerExpr`](https://github.com/lysu/tidb/blob/source-read-prepare/types/parser_driver/value_expr.go#L167) 会重写为 Constant 类型的 expression，但如果该条 stmt 支持 Cache 的话会重写为 Constant 并带上一个特殊的 `DeferredExpr` 指向一个 [`GetParam`](https://github.com/lysu/tidb/blob/source-read-prepare/expression/builtin_other.go#L787) 的函数表达式，而这个函数会在执行时实际从前面 Execute 保存到 [`sessionVars.PreparedParams`](https://github.com/lysu/tidb/blob/source-read-prepare/sessionctx/variable/session.go#L190) 中获取，这样就做到了 Plan 并 Cache 一个参数无关的 Plan，然后实际执行的时填充参数。

新获取 Plan 后会保存到 `preparedPlanCache` 供后续使用。

### 命中 Cache

让我们回到 [`getPhysicalPlan`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/core/common_plans.go#L188)，如果 Cache 命中在获取 Plan 后我们需要重新 build plan 的 range，因为前面我们保存的 Plan 是一个带 `GetParam` 的函数表达式，而再次获取后，当前参数值已经变化，我们需要根据当前 Execute 的参数来重新修正 range，这部分逻辑代码位于 [`Execute#rebuildRange`](https://github.com/lysu/tidb/blob/source-read-prepare/planner/core/common_plans.go#L214) 中，之后就是正常的执行过程了。

## 文本协议的 Prepared

前面主要介绍了二进制协议的 Prepared 执行流程，还有一种执行方式是通过二进制协议来执行。

客户端可以通过 `COM_QUREY` 发送：

```
PREPARE stmt_name FROM prepareable_stmt;
EXECUTE stmt_name USING @var_name1, @var_name2,...
DEALLOCTE PREPARE stmt_name
```

来进行 Prepared，TiDB 会走正常 [文本 Query 处理流程](https://zhuanlan.zhihu.com/p/35134962)，将 SQL 转换 Prepare，Execute，Deallocate 的 Plan， 并最终转换为和二进制协议一样的 `PrepareExec`，`ExecuteExec`，`DealocateExec` 的执行器进行执行。

## 写在最后

Prepared 是提高程序 SQL 执行效率的有效手段之一。熟悉 TiDB 的 Prepared 实现，可以帮助各位读者在将来使用 Prepared 时更加得心应手。另外，如果有兴趣向 TiDB 贡献代码的读者，也可以通过本文更快的理解这部分的实现。
