---
title: TiDB 助力东南亚领先电商 Shopee 业务升级
author: ['刘春辉','洪超']
date: 2018-12-25
summary: 截至目前，系统已经平稳运行了六个多月，数据量增长至 35TB，经历了两次扩容后现在集群共包含 42 个节点。
tags: ['互联网']
category: case
url: /cases-cn/user-case-shopee/
weight: 6
logo: /images/blog-cn/customers/shopee-logo.png
---


>作者介绍：**刘春辉**，Shopee DBA；**洪超**，Shopee DBA。

## 一、业务场景

[Shopee](https://shopee.com/) 是东南亚和台湾地区领先的电子商务平台，覆盖新加坡、马来西亚、菲律宾、印度尼西亚、泰国、越南和台湾等七个市场。Shopee 母公司 [Sea](https://seagroup.com/) 为首家在纽约证券交易所上市的东南亚互联网企业。2015 年底上线以来，Shopee 业务规模迅速扩张，逐步成长为区域内发展最为迅猛的电商平台之一：

* 截止 2018 年第三季度 Shopee APP 总下载量达到 1.95 亿次，平台卖家数量超过 700 万。

* 2018 年第一季度和第二季度 GMV 分别为 19 亿美金和 22 亿美金，2018 上半年的 GMV 已达到 2017 全年水平。2018 年第三季度 GMV 达到了创纪录的 27 亿美元, 较 2017 年同期年增长率为 153%。

* 2018 年双 11 促销日，Shopee 单日订单超过 1100 万，是 2017 年双 11 的 4.5 倍；刚刚过去的双 12 促销日再创新高，实现单日 1200 万订单。

![图 1](https://upload-images.jianshu.io/upload_images/542677-3c3531cbec9e73a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图 1 Shopee 电商平台展示图</center>

**我们从 2018 年初开始调研 TiDB，6 月份上线了第一个 TiDB 集群。到目前为止我们已经有两个集群、60 多个节点在线运行，主要用于以下 Shopee 业务领域：**

* 风控系统：风控日志数据库是我们最早上线的一个 TiDB 集群，稍后详细展开。

* 审计日志系统：审计日志数据库存储每一个电商订单的支付和物流等状态变化日志。

本文将重点展开风控日志数据库选型和上线的过程，后面也会约略提及上线后系统扩容和性能监控状况。

## 二、选型：MySQL 分库分表 vs TiDB

![图 2](https://upload-images.jianshu.io/upload_images/542677-da0953476c7a913c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图 2 风控日志收集和处理示意图</center>

风控系统基于大量历史订单以及用户行为日志，以实时和离线两种方式识别平台上的异常行为和欺诈交易。它的重要数据源之一是各种用户行为日志数据。最初我们将其存储于 MySQL 数据库，并按照 USER_ID 把数据均分为 100 个表。随着 Shopee 用户活跃度见长，数据体积开始疯长，到 2017 年底磁盘空间显得十分捉襟见肘了。作为应急措施，我们启用了 InnoDB 表透明压缩将数据体积减半；同时，我们把 MySQL 服务器磁盘空间从 2.5TB 升级到了 6TB。这两个措施为后续迁移 MySQL 数据到 TiDB 多争取了几个月时间。

关于水平扩容的实现方案，当时内部有两种意见：MySQL 分库分表和直接采用 TiDB。

### 1. MySQL 分库分表

* 基本思路：按照 USER_ID 重新均分数据（Re-sharding），从现有的 100 个表增加到1000 个甚至 10000 个表，然后将其分散到若干组 MySQL 数据库。

* 优点：继续使用 MySQL 数据库 ，不论开发团队还是 DBA 团队都驾轻就熟。

* 缺点：业务代码复杂度高。Shopee 内部若干个系统都在使用该数据库，同时我们还在使用 Golang 和 Python 两种编程语言，每一个系统都要改动代码以支持新的分库分表规则。

### 2. 直接采用 TiDB

* 基本思路：把数据从 MySQL 搬迁至 TiDB，把 100 个表合并为一个表。

* 优点：数据库结构和业务逻辑都得以大幅简化。TiDB 会自动实现数据分片，无须客户端手动分表；支持弹性水平扩容，数据量变大之后可以通过添加新的 TiKV 节点实现水平扩展。理想状况下，我们可以把 TiDB 当做一个「无限大的 MySQL」来用，这一点对我们极具诱惑力。

* 缺点：TiDB 作为新组件首次引入 Shopee 线上系统，我们要做好「踩坑」的准备。

最后，我们决定采用 TiDB 方案，在 Shopee 内部做「第一个吃螃蟹的人」。风控日志数据库以服务离线系统为主，只有少许在线查询；这个特点使得它适合作为第一个迁移到 TiDB 的数据库。

## 三、上线：先双写，后切换

我们的上线步骤大致如下：

* 应用程序开启双写：日志数据会同时写入 MySQL 和 TiDB。

* 搬迁旧数据：把旧数据从 MySQL 搬到 TiDB，并完成校验确保新旧数据一致。

* 迁移只读流量：应用程序把只读流量从 MySQL 逐步迁移至 TiDB（如图 3 所示）。

* 停止双写：迁移过程至此结束。

![图 3](https://upload-images.jianshu.io/upload_images/542677-3cad026a752152e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图 3 迁移过程图：保持双写，逐步从读 MySQL 改为读 TiDB</center>

双写方式使得我们可以把整个切换过程拖长至几个月时间。这期间开发团队和 DBA 团队有机会逐步熟悉新的 TiDB 集群，并充分对比新旧数据库的表现。理论上，在双写停掉之前，若新的 TiDB 集群遭遇短时间内无法修复的问题，则应用程序有可能快速回退到 MySQL。

除此之外，采用双写方式也让我们有了重构数据库设计的机会。这一次我们就借机按照用户所属地区把风控日志数据分别存入了七个不同的逻辑数据库：rc_sg，rc_my，rc_ph，…，rc_tw。Shopee 用户分布于七个不同地区。迁移到 TiDB 之前，所有日志数据共存于同一个逻辑数据库。按照地区分别存储使得我们能够更为方便地为每个地区的日志定制不同的数据结构。

## 四、硬件配置和水平扩容

上线之初我们一共从 MySQL 迁移了大约 4TB 数据到 TiDB 上。当时 TiDB 由 14 个节点构成，包括 3 个 PD 节点，3 个 SQL 节点和 8 个 TiKV 节点。服务器硬件配置如下：

* TiKV 节点
    * CPU: 2 * Intel(R) Xeon(R) CPU E5-2640 v4 @ 2.40GHz, 40 cores
    * 内存: 192GB
    * 磁盘: 4 * 960GB Read Intensive SAS SSD Raid 5
    * 网卡: 2 * 10gbps NIC Bonding
* PD 节点和 SQL 节点
    * CPU: 2 * Intel(R) Xeon(R) CPU E5-2640 v4 @ 2.40GHz, 40 cores
    * 内存: 64GB
    * 磁盘: 2 * 960GB Read Intensive SAS SSD Raid 1
    * 网卡: 2 * 10gbps NIC Bonding

**截至目前，系统已经平稳运行了六个多月，数据量增长至 35TB（如图 4 所示），经历了两次扩容后现在集群共包含 42 个节点。**

![图 4](https://upload-images.jianshu.io/upload_images/542677-c7c823c8f47b59a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图 4 风控日志 TiDB 数据库存储容量和使用状况</center>

### 性能

![图 5](https://upload-images.jianshu.io/upload_images/542677-1a1e97fa0e7d6ed9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图 5 风控日志 TiDB 数据库 QPS Total 曲线</center>

风控日志数据库的日常 QPS（如图 5 所示）一般低于每秒 20K，在最近的双 12 促销日我们看到峰值一度攀升到了每秒 100K 以上。

**尽管数据量较之 6 个月前涨了 8 倍，目前整个集群的查询响应质量仍然良好，大部分时间 pct99 响应时间（如图 6 所示）都小于 60ms。对于以大型复杂 SQL 查询为主的风控系统而言，这个级别的响应时间已经足够好了。**

![图 6](https://upload-images.jianshu.io/upload_images/542677-1487349092fa4fa8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图 6 风控日志 TiDB 数据库两天 pct99 查询响应时间曲线</center>

## 五、问题和对策

* TiDB 的字符串匹配区分大小写（Case Sensitive）。目前尚不支持 Case Insensitive 方式。应用程序做了适配以实现 Case Insensitive 方式的字符串匹配。

* TiDB 对于 MySQL 用户授权 SQL 语法的兼容支持尚不完善。例如，目前不支持 SHOW CREATE USER 语法，有时候不得不读取系统表（mysql.user）来查看一个数据库账户的基本信息。

* 添加 TiKV 节点后需要较长时间才能完成数据再平衡。据我们观察，1TB 数据大约需要 24 个小时才能完成拷贝。因此促销前我们会提前几天扩容和观察数据平衡状况。

* TiDB  v1.x 版本以 region 数目为准在各个 TiKV 节点之间平衡数据。不过每个 region 的大小其实不太一致。这个问题导致不同 TiKV 节点的磁盘空间使用率存在明显差异。据说新的 TiDB v2.x 对此已经做了优化，我们未来会尝试在线验证一下。

* TiDB v1.x 版本需要定期手动执行 Analyze Table 以确保元信息准确。PingCAP 的同学告诉我们说：当 (Modify_count / Row_count) 大于 0.3 就要手动 Analyze Table 了。v2.x 版本已经支持自动更新元数据了。我们后续会考虑升级到新版本。


## 六、未来规划

过去一年亲密接触之下，我们对 TiDB 的未来充满信心，相信 TiDB 会成为 Shopee 数据库未来实现弹性水平扩容和分布式事务的关键组件。当前我们正在努力让更多 Shopee 业务使用 TiDB。

我们规划把 Shopee 数据从 MySQL 迁移到 TiDB 上的路线是「先 Non-transactional Data（非交易型数据），后 Transactional Data（交易型数据）」。目前线上运行的集群都属于 Non-transactional Data，他们的特点是数据量超大（TB 级别），写入过程中基本不牵涉数据库事务。接下来我们会探索如何把一些 Transactional Data 迁移到 TiDB 上。

MySQL Replica 是另一个工作重点。MySQL Replica 指的是把 TiDB 作为 MySQL 的从库，实现从 MySQL 到 TiDB 实时复制数据。我们最近把订单数据从 MySQL 实时复制到 TiDB。后续来自 BI 系统以及部分对数据实时性要求不那么高的只读查询就可以尝试改为从 TiDB 读取数据了。这一类查询的特点是全表扫描或者扫描整个索引的现象较多，跑在 TiDB 可能比 MySQL 更快。当然，BI 系统也可以借助 TiSpark 绕过 SQL 层直接读取 TiKV 以提升性能。

目前我们基于物理机运行 TiDB 集群，DBA 日常要耗费不少精力去照顾这些服务器的硬件、网络和 OS。我们有计划把 TiDB 搬到 Shopee 内部的容器平台上，并构建一套工具实现自助式资源申请和配置管理，以期把 DBA 从日常运维的琐碎中解放出来。

## 七、致谢

感谢 PingCAP 的同学一年来对我们的帮助和支持。每一次我们在微信群里提问，都能快速获得回应。官方架构师同学还不辞辛劳定期和我们跟进，详细了解项目进度和难点，总是能给出非常棒的建议。

PingCAP 的文档非常棒，结构层次完整清晰，细节翔实，英文文档也非常扎实。一路跟着读下来，受益良多。

TiDB 选择了 Golang 和 RocksDB，并坚信 SSD 会在数据库领域取代传统机械硬盘。这些也是 Shopee 技术团队的共识。过去几年间我们陆续把这些技术引入了公司的技术栈，在一线做开发和运维的同学相信都能真切体会到它们为 Shopee 带来的改变。
