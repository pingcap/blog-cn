---
title: TiDB 在 Ping++ 金融聚合支付业务中的实践
author: ['宋涛']
date: 2018-02-26
summary: TiDB 具备了出色的分布式事务能力，完全达到了 HTAP 的级别。TiKV 基于 Raft 协议做复制，保证多副本数据的一致性，可以秒杀当前主流的 MyCat、DRDS 分布式架构，且数据库的可用性更高。
tags: ['金融']
category: case
url: /cases-cn/user-case-ping++/
aliases: ['/blog-cn/user-case-ping++/','/blog-cn/tidb-ping++/']
weight: 2
logo: /images/blog-cn/customers/ping++-logo.png
---

## Ping++ 介绍

Ping++ 是国内领先的支付解决方案 SaaS 服务商。自 2014 年正式推出聚合支付产品，Ping++ 便凭借“7 行代码接入支付”的极致产品体验获得了广大企业客户的认可。

如今，Ping++ 在持续拓展泛支付领域的服务范围，旗下拥有聚合支付、账户系统、商户系统三大核心产品，已累计为近 25000 家企业客户解决支付难题，遍布零售、电商、企业服务、O2O、游戏、直播、教育、旅游、交通、金融、房产等等 70 多个细分领域。

Ping++ 连续两年入选毕马威中国领先金融科技 50 强，并于 2017 成功上榜 CB Insights 全球 Fintech 250 强。从支付接入、交易处理、业务分析到业务运营，Ping++ 以定制化全流程的解决方案来帮助企业应对在商业变现环节可能面临的诸多问题。

## TiDB 在 Ping++ 的应用场景 - 数据仓库整合优化

Ping++ 数据支撑系统主要由流计算类、报表统计类、日志类、数据挖掘类组成。其中报表统计类对应的数据仓库系统，承载着数亿交易数据的实时汇总、分析统计、流水下载等重要业务:

![](http://upload-images.jianshu.io/upload_images/542677-1c557b3068be5b7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


随着业务和需求的扩展，数仓系统历经了多次发展迭代过程：

1.  由于业务需求中关联维度大部分是灵活多变的，所以起初直接沿用了关系型数据库 RDS 作为数据支撑，数据由自研的数据订阅平台从 OLTP 系统订阅而来。

2.  随着业务扩大，过大的单表已不足以支撑复杂的查询场景，因此引入了两个方案同时提供数据服务：ADS，阿里云的 OLAP 解决方案，用来解决复杂关系型多维分析场景。ES，用分布式解决海量数据的搜索场景。

3.  以上两个方案基本满足业务需求，但是都仍存在一些问题：

    + ADS：一是数据服务稳定性，阿里云官方会不定期进行版本升级，升级过程会导致数据数小时滞后，实时业务根本无法保证。二是扩容成本，ADS 为按计算核数付费，如果扩容就必须购买对应的核数，成本不是那么灵活可控。

    + ES：单业务搜索能力较强，但是不适合对复杂多变的场景查询。且研发运维代价相对较高，没有关系型数据库兼容各类新业务的优势。

所以需要做出进一步的迭代整合，我们属于金融数据类业务，重要性安全性不能忽视、性能也得要有保障，经过我们漫长的调研过程，最终，由 PingCAP 研发的 TiDB 数据库成为我们的目标选型。

TiDB 具备的以下核心特征是我们选择其作为实时数仓的主要原因：

+ 高度兼容 MySQL 语法；

+ 水平弹性扩展能力强；

+ 海量数据的处理性能；

+ 故障自恢复的高可用服务；

+ 金融安全级别的架构体系。

并追踪形成了以下数据支撑系统架构：

![](http://upload-images.jianshu.io/upload_images/542677-0a6aa279544b9141.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

新的方案给我们的业务和管理带来了以下的提升和改变:

+ 兼容：整合了现有多个数据源，对新业务上线可快速响应；

+ 性能：提供了可靠的交易分析场景性能；

+ 稳定：更高的稳定性，方便集群运维；

+ 成本：资源成本和运维成本都有所降低。

## TiDB 架构解析及上线情况

TiDB 是 PingCAP 公司受 Google Spanner / F1 论文启发而设计的开源分布式 NewSQL 数据库。从下图 Google Spanner 的理念模型可以看出，其设想出数据库系统把数据分片并分布到多个物理 Zone 中、由 Placement Driver 进行数据片调度、借助 TrueTime 服务实现原子模式变更事务，从而对外 Clients 可以提供一致性的事务服务。因此，一个真正全球性的 OLTP & OLAP 数据库系统是可以实现的。

![](http://upload-images.jianshu.io/upload_images/542677-9c9f91a097eb5a16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们再通过下图分析 TiDB 整体架构：

![](http://upload-images.jianshu.io/upload_images/542677-ae6b78de27f9142e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看出 TiDB 是 Spanner 理念的一个完美实践，一个 TiDB 集群由 TiDB、PD、TiKV 三个组件构成。

+ TiKV Server：负责数据存储，是一个提供事务的分布式 Key-Value 存储引擎；

+ PD Server：负责管理调度，如数据和 TiKV 位置的路由信息维护、TiKV 数据均衡等；

+ TiDB Server：负责 SQL 逻辑，通过 PD 寻址到实际数据的 TiKV 位置，进行 SQL 操作。

生产集群部署情况：

![](http://upload-images.jianshu.io/upload_images/542677-aef872b2131e3cb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现已稳定运行数月，对应的复杂报表分析性能得到了大幅提升，替换 ADS、ES 后降低了大量运维成本。


## TiDB 在 Ping++ 的未来规划


1. TiSpark 的体验

    TiSpark 是将 Spark SQL 直接运行在分布式存储引擎 TiKV 上的 OLAP 解决方案。下一步将结合 TiSpark 评估更加复杂、更高性能要求的场景中。

2. OLTP 场景

    目前数仓 TiDB 的数据是由订阅平台订阅 RDS、DRDS 数据而来，系统复杂度较高。TiDB 具备了出色的分布式事务能力，完全达到了 HTAP 的级别。

    TiKV 基于 Raft 协议做复制，保证多副本数据的一致性，可以秒杀当前主流的 MyCat、DRDS 分布式架构。且数据库的可用性更高，比如我们对生产 TiDB 集群所有主机升级过磁盘（Case 记录），涉及到各个节点的数据迁移、重启，但做到了相关业务零感知，且操作简单，过程可控，这在传统数据库架构里是无法轻易实现的。

    我们计划让 TiDB 逐渐承载一些 OLTP 业务。

## 对 TiDB 的建议及官方回复


1. DDL 优化：目前 TiDB 实现了无阻塞的 online DDL，但在实际使用中发现，DDL 时生成大量 index KV，会引起当前主机负载上升，会对当前集群增加一定的性能风险。其实大部分情况下对大表 DDL 并不是很频繁，且时效要求并不是特别强烈，考虑安全性。建议优化点：

    + 是否可以通过将源码中固定数值的 defaultTaskHandleCnt、defaultWorkers 变量做成配置项解决；

    + 是否可以像 pt-osc 工具的一样增加 DDL 过程中暂停功能。

2. DML 优化：业务端难免会有使用不当的 sql 出现，如导致全表扫描，这种情况可能会使整个集群性能会受到影响，对于这种情况，是否能增加一个自我保护机制，如资源隔离、熔断之类的策略。

针对以上问题，我们也咨询了 TiDB 官方技术人员，官方的回复如下：

+ 正在优化 Add Index 操作的流程，降低 Add Index 操作的优先级，优先保证在线业务的操作稳定进行。

+ 计划在 1.2 版本中增加动态调节 Add Index 操作并发度的功能。

+ 计划在后续版本中增加 DDL 暂停功能。

+ 对于全表扫描，默认采用低优先级，尽量减少对于点查的影响。后续计划引入 User 级别的优先级，将不同用户的 Query 的优先级分开，减少离线业务对在线业务的影响。

最后，特此感谢 PingCAP 所有团队成员对 Ping++ 上线 TiDB 各方面的支持！


✎ 作者：宋涛，Ping++ DBA



