---
title: What’s New in TiDB 4.0.0
author: ['段兵']
date: 2020-03-12
summary: TiDB 4.0 版本是一个让人兴奋的版本，我们提供了非常多有用、有趣的特性，本文将带领大家对这些有用、有趣的特性作一个概括性的讲述。
tags: ['TiDB 4.0 新特性','社区']
---

2020 年 2 月 28 日，我们发布了 TiDB 4.0.0 beta-1 版本，TiDB 4.0 版本是一个让人兴奋的版本，我们提供了非常多有用、有趣的特性，例如提前剧透的 [Key Visualizer](https://pingcap.com/blog-cn/tidb-4.0-key-visualizer/)、[BR](https://pingcap.com/blog-cn/cluster-data-security-backup/)。为了让大家有一个全局的了解，本文将带领大家对这些有用、有趣的特性作一个概括性的讲述，后续我们还会针对每一个特性单独进行解读。

## 新增特性解读

### 系统管理

4.0 版本中我们提供一个简洁的 Dashboard，通过 Dashboard 可查看集群拓扑和运行状态、运行参数、系统日志、集群是否有常见问题和异常指标、基础性能指标等，极大提升 DBA 同学了解系统状态、排查问题的效率。

4.0 版本中我们将系统的配置项全部移到内核系统表中，并且可能通过 SQL 查看、修改相关的配置项，极大的提升 DBA 同学运维 TiDB 的效率。

4.0 版本中我们提供一个新的部署、运维工具 TiOps，新工具可以按组件或者角色部署、重启、停止、版本升级等功能，能在 10min 以内完成一个测试集群的搭建，极大的提升运维同学部署、运维 TiDB 的效率，[TiOps 操作演示视频](https://v.qq.com/x/page/z0932k1gtj6.html)。

4.0 版本中我们提供一个包管理器的工具 TiUP，管理 TiDB 生态中的所有组件。TiUP 非常适合想要快速试用、体验 TiDB 功能的场景。例如：在本地机器部署集群、部署自行编译的新版本等场景。极大的方便了喜欢尝鲜的用户，[TiUP 操作演示视频](https://v.qq.com/x/page/o0932h32n8z.html)。

### 悲观事务模型、支持 10GB 的大事务

悲观事务模型是作为关系型数据库关键核心特性，适合冲突多的业务场景，避免了事务在提交时才发现冲突而回滚，浪费大量执行时间。在兼容性上，悲观事务模型也被众多传统主流数据库所采用。因此，支持了悲观事务的 TiDB 能让传统行业用户更加平滑得从其他数据库进行迁移。例如：金融行业中的银行核心账务系统等。在 4.0 版本中我们将交付此功能，满足金融行业业务的需求。

在 4.0 版本之前，TiDB 的事务大小限制为 100MB，当事务写入量超过时将回滚该事务。对于传统数据库用户，需要在业务系统触及此限制时修改业务逻辑，分批进行事务提交，给业务迁移带来了困难。在 4.0 版本中，TiDB 的事务进行了大量优化，可以支持最大 10GB 的事务。相信在大多数业务场景中，都可以满足用户对事务容量的需求，降低业务迁移的难度。

### 全新的热点调度框架、策略，全新的弹性调度算法，全新打造基于规则的副本调度系统

对于互联网应用的开发同学来说热点数据、访问热点并不陌生而且也经常遇到。在 4.0 版本中我们实现了一套强大、多维度的调度框架，并基于框架实现了根据访问流量、单个 Key-Value 访问流量、热点 Key-Value 的个数三种统计信息的调度算法，在很大程度上解决了 TiDB 的热点问题。

TiDB 产品天生为云而生，在 4.0 中我们针对部分公有云厂商底层基础设施特别设计一套强大的弹性调度系统，充分利用公有云厂商按需分配、秒级交付的能力，根据系统的负载情况，自适应的为用户分配资源，提升资源的利用率，极大降低用户的成本。

4.0 版本中我们实现了一套灵活、强大的副本规则系统供调度系统使用。通过配置不同的副本规则系统，满足用户不同的调度需求。例如：配置副本位置策略满足业务就近访问数据的需求。通过规则可以控制任意一段（Region）数据的副本数、副本地理位置策略、副本存储主机的类型、是否参与 Raft 协议的投票、是否承担 Raft Leader 角色等。

### 列存储引擎（TiFlash）

[TiFlash](https://pingcap.com/blog-cn/10x-improving-analytical-processing-ability-of-tidb-with-tiflash/) 是 TiDB 的列存引擎。列存储引擎、行存储引擎可保持数据实时同步且保持数据的一致性。列存储引擎、行存储引擎可分别部署，提供物理级别的资源隔离，为确保系统稳定性提供坚实的基础，为 TiDB 的 HTAP 形态提供了关键支持。用户可以很方便地在 TiDB 集群加入 TiFlash 并依赖 TiDB 优化器自动获得查询加速，也可以在交易场景下无干扰地查询最新的在线数据。

### 丰富的 SQL 功能

众所周知，SQL 执行计划的的好坏是影响 SQL 执行性能的关键因素之一，而 SQL 执行计划是否稳定是影响 SQL 执行的延时是否会波动的关键因素之一。SQL 执行计划的好坏受统计信息准确性、统计信息的变化等诸多因素影响，因此执行计划会发生预期外的变化，导致响应时间过长或者响应时间波动比较大。此种情况用户可以通过 SQL Hint 来手动指导选择执行计划，以确保系统的稳定性。但 SQL Hint 需要修改 SQL 文本、修改业务代码，使用成本较高。SPM（SQL Plan Management）为此而生，SPM 结合 SQL Binding、SQL Hint 功能自己选择、创建、绑定系统认为相对较优的执行计划。

基于代价（CBO）的优化器在绝大数场景会选择相对较优的执行计划，但在部分场景优化器受统计信息准确性、统计信息的变化等诸多因素影响先择的不是相对较优的执行计划。此时需要人手工干预优化器并采用指定的数据读取方式，连接类型生成执行计划，以此达到提升 SQL 语句执行效率的目的。在 4.0 版本中我们提供 15 种 Hint 来帮助大家人手工干预优化器。

系统 OOM 在 4.0 之前一直困扰着我们的用户，在 4.0 中我们将保留 Index Join 通过批量读取数据到内存的同时，将 IndexJoin 的过程拆分成 IndexHashJoin、IndexMergeJoin 两个过程，通过拆分我们能比较好的解决 Join 过程中系统 OOM 的问题。当然在 Join 过程中若是数据量比较多的情况在 4.0 中我们也提供了将 Join 的中间结果写入磁盘的功能，通过写磁盘来避免系统出现 OOM 的情况。

表达式索引功能，也称为函数式索引，函数式索引能在一个表达式上建立索引，在某些情况下提升 SQL 的性能。

Index Merge 功能，Index Merge 功能涉及到多个索引的返回结果进行合并，在某些情况下提升 SQL 的性能。

Sequence Generator 是 SQL 2003 标准中加入的数据库特性，通过 Sequence 系统可自动生成连续的数值供其他模块使用，Sequence 与 Auto_Increment 相比更加灵活，适用场景更广。

在创建整型的主键或者唯一索引时，给列添加 `auto_random` 属性，系统就会自动随机填充数据，解决通过自增列做主键或者唯一索引时引起的热点问题。

支持添加、删除主键的功能。

采用 UTF8MB4 为默认字符集，同时也支持大小写敏感的 Collation（`utf8mb4_general_ci`）。

### CDC

CDC（Change Data Capture）工具用于捕捉 TiDB 上的数据变更数据并将数据交付给下游系统，例如：TiDB、MySQL、消息队列、分布式文件系统等。相比于 TiDB Binlog，不再依赖 TiDB 事务模型保证数据同步的一致性，系统可水平扩展且天然提供高可用的特性。

### Follower Read

4.0 中我们提供了 Follower Read 功能，此功能可以将 Region Leader 的读压力转移一部分到 Region 的 Follower 上，这样就可以减轻  Region Leader 的压力，提升整个系统的性能，充分利用系统资源。

### 系统的可观测性

4.0 中我们提供 Statement Summary 系统表将相似的 SQL 及相关的执行计划汇聚在一组，并统计执行过程中各个阶段的各项性能指标，用户可以通过 SQL 查询  Statement Summary 系统表或者通 Statements UI 查看统计信息、分析、发掘有性能问题的 SQL 语句。

4.0 中我们提供内置 SQL 性能诊断工具帮助 DBA 同学诊断系统的性能问题，内置工具可以通过 SQL 来访问，工具将集群拓扑信息、硬件信息、软件信息、内核参数、性能指标、系统信息、慢查询日志、Statements 等信息汇聚在一起，并基于简单的规则判断帮助 DBA 排查问题。

热点问题的排查在 4.0 之前只能通过系统的表现出的特征逐渐搞清楚热点数据的位置、产生热的原因等，期间涉及检查多个组件的 CPU 和 IO 是否均衡、根据热点 Region 列表逐一检查热点表、根据表分析业务逻辑等等。在 4.0 中我们提供了一个 [Key Visualizer](https://pingcap.com/blog-cn/tidb-4.0-key-visualizer/) 的功能，很轻松地给集群拍个 “CT”，快速直观地观察集群整体热点及流量分布情况。

### 备份与恢复

4.0 版本中我们通过 [BR（Backup&Restore）](https://pingcap.com/blog-cn/cluster-data-security-backup/) 工具提供快速备份与恢复数据的功能，备份与恢复的性能高达 1GB/秒，再也不用担心因为数据量过大，无法完成备份与恢复，从根本上解决删库跑路的问题。当研发、DBA 不小心通过 Truncate Table 将数据删除时，我们也贴心的提供 flashback 命令快速恢复被删除的数据。


### 安全

4.0 版本中我们提供了动态更新各个组件 TLS 证书的功能，解决了 TLS 证书到期系统需要重新生成、更新证书并重启服务的问题。

4.0 版本中我们提供磁盘文件静态加密功能。

### 性能

与 3.0 版本相比 TPC-C 提升 50% 。性能的提升主要归功于提供 New Row Format 格式、Full Vectorized Expression Evaluation、Unified Thread Pool 等功能。

## 开源社区

4.0 版本的开发过程中，社区依然给我们很有力的支持，在这里对各位贡献者表示由衷的感谢。如下：

*  [PR/14942](https://github.com/pingcap/tidb/pull/14942),[PR/14919](https://github.com/pingcap/tidb/pull/14919), [PR/14696](https://github.com/pingcap/tidb/pull/14696):  [gauss1314](https://github.com/gauss1314) and [hsqlu](https://github.com/hsqlu) helped to refine the output of `explain`.

*   [PR/14600](https://github.com/pingcap/tidb/pull/14600):  [hsqlu](https://github.com/hsqlu) helped to accomplish spilling intermediate results to disk when the execution engine exceeds the memory protection ratio.

*   [PR/10512](https://github.com/pingcap/tidb/pull/10512), [PR/12305](https://github.com/pingcap/tidb/pull/12305) and [Issue/14332](https://github.com/pingcap/tidb/issues/14332): [hailanwhu](https://github.com/hailanwhu) and [sduzh](https://github.com/sduzh) together supported the `Index Merge` feature. Enables TiDB SQL Engine to use more than one index to improve the performance and robustness of query processing.

*   [PR/14458](https://github.com/pingcap/tidb/pull/14458):  [catror](https://github.com/catror) vectorized the `Merge Join` executor which greatly improved the execution performance. It gets 57% faster than before in 4 threads.

*   [PR/14238](https://github.com/pingcap/tidb/pull/14238): [pingyu](https://github.com/pingyu) optimized the execution performance of Window Function,

*   [TiKV #5725](https://github.com/tikv/tikv/pull/5725): @niedhui Implemented the new row format at TiKV side.

*   [TiKV #6685](https://github.com/tikv/tikv/pull/6685), [TiKV #6592](https://github.com/tikv/tikv/pull/6592), [TiKV #6713](https://github.com/tikv/tikv/pull/6713): @TennyZhuang Added collation support for index executor and built-in functions at TiKV side.

*   [TiKV #5866](https://github.com/tikv/tikv/pull/5866), [TiKV #6000](https://github.com/tikv/tikv/pull/6000): @TennyZhuang greatly improved the performance of several built-in functions at TiKV side.

## Quick Start

### TiUP Quick Start

下载安装包

```
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

快速部署本地集群

```
tiup playground nightly
```

试用 TiDB

```
mysql -h localhost -P 4000 -u root
```

或

```
tiup client
```

###  TiOps Quick Start

下载安装 rpm 包

```
wget https://download.pingcap.org/tiops-v0.2.0-2.el7.x86_64.rpm
```

安装 rpm 包

```
sudo yum -y localinstall tiops-v0.2.0-2.el7.x86_64.rpm
```

编辑拓扑配置

```
拷贝模块：cp /usr/share/tiops/topology.yaml.example topology.yaml
编辑拓扑文件：vim topology.yaml
```

快速部署

```
tiops quickdeploy -c demo -d tidb -u root -T topology.yaml -t 4.0.0-beta.1 -f 5
```

试用 TiDB

```
mysql -h <tidb-server-ip> -P 4000 -u root
```

## One More Thing

TiDB 4.0 RC 版本将于 2020 年 3 月底正式发布，计划 2020 年 5 月底正式 GA，欢迎大家试用。
