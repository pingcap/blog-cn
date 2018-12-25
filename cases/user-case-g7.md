---
title: TiDB 在 G7 的实践和未来
author: ['廖强']
date: 2018-01-15
summary: 2018 年初，运维团队和每一个业务方进行了一次需求沟通，业务方对 TiDB 的需求越来越强烈。我们准备让TiDB发挥应用到更多的场景中。
tags: ['互联网']
category: case
url: /cases-cn/user-case-g7/
aliases: ['/blog-cn/user-case-g7/']
weight: 19
logo: /images/blog-cn/customers/g7-logo.png
---

## 背景

2010 年，G7 正式为物流运输行业提供面向车队管理的 SaaS 服务，经过持续创新，通过软硬一体化的产品技术能力，致力于数字化每一辆货车，以实时感知技术创造智慧物流新生态。G7 为客户提供全方位的数据服务、智能的安全和运营管理、手机管车、数字运力、以及 ETC、油和金融等增值服务。

目前，G7 连接了 600,000 辆货车，每天行驶 6500 万公里（可绕地球赤道 1625 圈），13.5 亿个轨迹点和 2,200 万次车辆事件触发，并且以直线速度飞速增长。G7 每天产生的车辆行驶、状态、消费等数据超过 2T，飞速增加的车辆、数据类型和复杂的金融业务，使得数据库的事务、分析、扩展和可用性面临巨大的挑战。

在大量的车辆信息和轨迹相关数据业务中，当前我们通过 Spark、Hive 等对大量原始数据进行分析后，存入阿里云 DRDS，对外提供基础数据接口服务。由于清洗后的数据量依然很大，使用 DRDS 的存储成本非常高，且面对很多 OLAP 的查询时，效率不如人意。

而在金融和支付这种复杂业务场景中，面临 CAP 中 C 和 P 的挑战。在以往的工作中，支付系统由于面临强一致性事务的高峰值写入问题，采用了 2PC+MySQLXA（单个 MySQL 作为参与者，上层增加 Proxy 作为协调者）完成了分布式事务数据库的方案。但是这种方案在 Partition 中，极为麻烦。同时，运营和风控系统经常需要做相对复杂的查询，要么通过 MySQL+ETL+OLAP 数据库（成本高），要么容忍查询的效率问题。


## 探索

G7 的技术团队一直在寻找一种能解决上述问题的数据库。要找到这样一种数据库，除了需要满足上述需求以外，还需要满足另一个需求：可维护性和易迁移性。这要求该数据库具体需要满足如下几个要求：

+ 兼容 MySQL 协议，使得数据库的变更对上层业务透明，这个有多重要，做过基础架构升级落地的同学应该深有感触。

+ 支持 MySQL 的主从同步机制，使得数据库的变更可以平滑逐步升级，降低变更风险。

+ 必须是开源的。数据库的稳定需要付出很大的精力和时间，在这个过程中，或多或少都出现问题。出现问题不可怕，可怕的是无法定位和解决问题，只能依赖“他人”。数据库的一个 BUG 对“他人”来说，可能是一个小问题，对 G7 业务而言，可能是一个巨大的灾难。当“屁股”不在同一个板凳上时，我们需要对数据库有很强的掌控力。

+ 开源的同时，背后一定需要有一个有实力的技术团队或商业公司的全力投入。在见识了不少“烂尾”和“政绩”的开源项目后，只有一个稳定全职投入的技术团队或商业公司，才能最终让这个数据库越来越好。

在这么多限制和需求的情况下，TiDB+TiSpark 很快进入我们的视线，并且开始调研。通过和 TiDB 技术人员的交流，除了满足上述的需求外，如下技术细节使我们一致认为可以选择这样的方案：

+ 将 MySQL 架构中 Server 和 StorageEngine 概念进一步松耦合，分为 TiDB 和 TiKV，水平扩展性进一步提升。

+ 定位于 Spanner 的开源实现，但是没有选择 Multi-Paxos，而是采用了更容易理解、实现和测试的 Raft，使得我们在分布式一致性上少了很多担忧。

+ 使用 RocksDB 作为底层的持久化KV存储，单机的性能和稳定性经过了一定的考验。

+ 基于 GooglePercolator 的分布式事务模型，在跨区域多数据中心部署时对网络的时延和吞吐要求比较高，但我们目前没有这样的强需求。

## 初体验——风控数据平台

该风控数据平台是将众多的业务数据做清洗和一定复杂度的计算，形成一个客户在 G7 平台上金融数据指标，供后续的风控人员来查询客户的风险情况，同时支撑运营相对复杂的查询。由于数据量较大，传统的关系型数据库在扩展性和处理 OLAP 上不符合该业务的需求；同时该业务面向内部，在一开始不熟悉的情况下，不会影响到客户，因此，我们决定在这个业务上，选择使用 TiDB。风控数据平台开始于 2017 年 8 月，2017 年 10 月上线了第一个版本，对线上用户提供服务。最开始使用的 TiDB RC4 版本，后升级到 Pre-GA，我们计划近期升级到 GA 版本。

系统架构如下所示，整个流程非常简洁高效。

![](http://upload-images.jianshu.io/upload_images/542677-0dcd862f658723cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在使用的过程中，我们还是遇到了不少兼容性相关的问题。为了增加我们对 TiDB 的理解，我们和 TiDB 技术团队取得联系，积极参与到 TiDB 项目中，熟悉代码和修复部分兼容性和 BUG 相关的问题。以下是我们在实践过程中解决的问题：

+ 修复 `INFORMATION_SCHEMA.COLUMNS` 中 ，`COLUMN_TYPE` 不支持 UNSIGNED 的兼容性问题。

    [https://github.com/pingcap/tidb/pull/3818](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/3818)

+ 修复 IGNORE 关键字对 INSERT、UPDATE和DELETE 的兼容性问题。

    [https://github.com/pingcap/tidb/pull/4376](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/4376)

    [https://github.com/pingcap/tidb/pull/4397](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/4397)

    [https://github.com/pingcap/tidb/pull/4564](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/4564)

+ 修复 Set 和 Join 中存在的 PanicBUG。

    [https://github.com/pingcap/tidb/pull/4326](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/4326)

    [https://github.com/pingcap/tidb/pull/4613](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/4613)

+ 增加了对 SQL_MODE 支持 ONLY_FULL_GROUP_BY 的特性。

    [https://github.com/pingcap/tidb/pull/4613](http://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb/pull/4613)

这里仍然存在一个与 MySQL 不兼容的地方。当开启事务后，如果 insert 的语句会导致主键或者唯一索引冲突时，TiDB 为了节省与 TiKV 之间的网络开销，并不会去 TiKV 查询，因此不会返回冲突错误，而是在 Commit 时才告知是不是冲突了。希望准备使用或关注 TiDB 的朋友能注意到这一点。后来我们咨询 TiDB 官方，官方的解释是：TiDB 采用乐观事务模型，冲突检测在执行 Commit 操作时才会进行。

感谢在初体验过程中，TiDB 团队非常认真、负责和快速的帮助我们排查和解决问题，提供了非常好的远程支持和运维建议。

## 在其它业务中的推广规划

2018 年初，运维团队和每一个业务方进行了一次需求沟通，业务方对 TiDB 的需求越来越强烈。我们正沿着如下的路径，让 TiDB 发挥应用到更多的场景中。

+ 将 TiDB 作为 RDS 的从库，将读流量迁移到 TiDB；

+ 从内部业务开始，逐步将写流量迁移到 TiDB；

+ 将更多 OLAP 的业务的迁到 TiSpark 上；

+ 合作开发 TiDB 以及 TiDB 周边工具。

## 参与 TiDB 社区的 Tips

+ 善用 GDB 工具去了解和熟悉 TiDB 的代码结构和逻辑。

+ 初始选择一些 Issue，去分析和尝试修复。

+ 利用火焰图去关注和优化性能。

+ 如果没有读过周边的论文，可以试着去读一读，加深对系统原理的理解。

+ 积极参与 TiDB 的社区活动，加深与 TiDB 核心研发的沟通。

+ 有合适的业务场景，可以多试用 TiDB，拓宽 TiDB在 不同场景下的应用实践。



> 作者简介：廖强，曾供职于百度，负责百度钱包的分布式事务数据库，基础架构和收银台。现 G7 汇通天下技术合伙人，负责金融产品研发、运维和安全。
