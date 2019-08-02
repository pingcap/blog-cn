---
title: TiDB 在平安核心系统的引入及应用
author: ['何志勇']
date: 2019-05-28
summary: 这个应用场景是我们的产险的实际分析场景，表数据量不大但是 SQL 较为复杂，是典型的星型查询。在 Oracle 用了 134 秒，但是 TiDB 用了 50 分钟，我们觉得很诧异，与 TiDB 的同事咨询后，他们通过现场支持我们优化底层代码后 34 秒可以跑出来。
tags: ['金融']
category: case
url: /cases-cn/user-case-pingankeji/
weight: 2
logo: /images/blog-cn/customers/pingankeji-logo.png
---


作者介绍：何志勇，平安科技数据库产品团队  资深工程师。

>本文转载自公众号「平安科技数据库产品团队」。
>2019 年 5 月 9 日，平安科技数据库产品资深工程师何志勇在第十届数据库技术大会 DTCC 上分享了《TiDB 在平安核心系统的引入及应用》，通过对 TiDB 进行 POC 测试，详细解析如何选择适用于金融行业级别的开源分布式数据库，以及平安“财神节”活动中引入 TiDB 的全流程应用实践案例分享。本文根据演讲内容整理。


## 一、TiDB 引入的 POC 测试

作为一名运维人员，引入一个新的数据库产品前必须要明确几点： 

*   从业务的角度，引入的产品能否满足业务基本需求和使用场景。

*   从运维管理角度看，这产品必须是可运维、可管理的，并且我们需要对其相应的功能与特性，要有一个很好的了解。

*   产品性能稳定。

所以在我们引入前从以下六个方面分别对 TiDB 进行测试验证，其中功能与架构、配置与管理、备份与恢复都是针对我们运维管理，SQL 特性、基准测试、应用场景测试则是应对业务需求和业务场景的。

![](media/user-case-pingankeji/1.PNG)

### 1. 功能与架构

TiDB 事务隔级别为 SI，支持 Spark 生态，支持动态扩容，跨数据中心部署。

这是 TiDB 官网最新的架构图：

![](media/user-case-pingankeji/2.JPG)

从左至右看，可以通过 MySQL 或 MySQL 客户端接入 TiDB，TiDB 有 TiDB、PD、TiKV 三个组件，组件之间功能相互独立，需独立部署，分别负责计算、调度、存储功能；同时又相互协作，共同完成用户请求处理。在 TiKV 层各节点是使用 Raft 协议保证节点间数据的一致性，同时它还提供 Spark 接口供大数据分析。 

从上往下看，可通过 Data Miaration 工具从 MySQL 迁移到 TiDB，同时提供备份恢复功能、内部性能监控监测及诊断、支持容器化部署。

**TiDB 从架构及生态上基本上具备了传统数据库应有的功能。**

### 2. SQL 特性

兼容 mysql 语法，2.0 版本不支持窗口函数、分区表、视图、trigger 等。

![](media/user-case-pingankeji/3.JPG)

![](media/user-case-pingankeji/4.JPG)

### 3. 配置与管理

支持在线 DDL，2.0 只支持串行的 DDL、不支持并发，在优化器上支持 RBO 与 CBO，能对单会话进行管理，可以支持复杂的 SQL。

![](media/user-case-pingankeji/5.JPG)

### 4. 备份与恢复

备份恢复工具均为开源，支持多线程备份恢复，当前版本不支持物理备份，loader 恢复时间偏长。

![](media/user-case-pingankeji/6.JPG)

### 5. 基准测试

TiDB 在单条 SQL 的性能较好，高并发场景下性能较稳定，但 DML 事务大小有限制。

![](media/user-case-pingankeji/7.JPG)

![](media/user-case-pingankeji/8.JPG)

![](media/user-case-pingankeji/9.JPG)

![](media/user-case-pingankeji/10.JPG)

![](media/user-case-pingankeji/11.JPG)

### 6. 应用场景测试

支持标量子查询，能支持非常复杂的查询，查询引擎可朔性强。

![](media/user-case-pingankeji/12.JPG)

![](media/user-case-pingankeji/13.JPG)

**这个应用场景是我们的产险的实际分析场景，表数据量不大但是 SQL 较为复杂，是典型的星型查询。在 Oracle 用了 134 秒，但是 TiDB 用了 50 分钟，我们觉得很诧异，与 TiDB 的同事咨询后，他们通过现场支持我们优化底层代码后 34 秒可以跑出来。**

## 二、“财神节”活动中 TiDB 的应用实战

“财神节”是中国平安综合性年度线上金融狂欢节。2019 年平安集团“财神节”活动于 1 月 8 日正式启动，涉及寿险、产险、银行、养老险、健康险、普惠、证券、基金、健康互联、陆金所、壹钱包、互娱、不动产等多个领域，活动参与的 BU 数量与推广的力度是历年之最。单日成交额超过 1000 亿，在单日交易额破千亿背后是几百个后台数据库实例的运维保障。

**我们看下活动业务场景的特点：**

*   **参与门槛低**：暖宝保这个业务保费价格低至 19.9，所以人人都可以参与。

*   **我们的推广力度很大**：以微服务的方式对接如平安健康、好福利、平安银行、陆金所等所有 APP 端，同时配合各种合作伙伴的宣传。

*   **典型的互联网活动形式：如秒杀、红包雨，所以对数据库的要求是高并发、低延迟、高响应、高可用，2-5 年在线数据存储量预计达到 20~50TB，而这些只是预估，有可能远远大于以上评估值。**

![](media/user-case-pingankeji/14.PNG)

平安在用的开源数据库有很多，那在这么多数据库中，我们选择什么数据库呢？

![](media/user-case-pingankeji/15.PNG)

综合对比考量最终我们选择 TiDB，在选择的同时也面临着挑战：

*   **时间紧迫**

    2018 年 12 月 17 日~2019 年 1 月 7 日，20 天时间内完成开发测试到生产上线，时间短，风险大

*   **开发零使用经验**

    现有开发大都是基于传统 Oracle 保险业务，对于 TiDB 没有使用经验

*   **并发量与扩容**

    互联网业务并发需求前期不可完全需求，前期不能很好的以实际压力进行测试，与资源准备

*   **DB 运维管理**

TiDB 还处于生产落地阶段，一类系统尚未使用过 TiDB，没有大规模应用运维经验

基于以上挑战，我们在 9 台 PC 服务器上做了验证测试，测试工具是 jmeter，TiKV 节点数我们是逐步增加的，具体的测试过程如下：

![](media/user-case-pingankeji/16.JPG)

![](media/user-case-pingankeji/17.JPG)

![](media/user-case-pingankeji/18.JPG)

![](media/user-case-pingankeji/19.JPG)

![](media/user-case-pingankeji/20.JPG)

![](media/user-case-pingankeji/21.JPG)


总结一下，就是：

*   **TiDB 吞吐**：在 select 中即 point select，TiDB 的吞吐比较好。

*   **弹性扩容**：在 insert 场景下随着节点数的增加，TPS 也会相应的增加，每增加 3 个节点 TPS 可提升 12%~20% 左右，同时在相同 TiKV 节点数下，TPS 与响应时间，此消彼长。

*   **批量提交性能尤佳**：业务中一个保单需要同时写 7 个表，7 个表同时 commit 比单表 commit TPS 高，相同 TPS 场景下延迟更小。

*   **初始化 region 分裂耗时长**：因在测试时没有预热数据（表为空表），对空表写入前几分钟，响应时间会比较大，约 5~8 分钟后响应时间趋于稳定。在前几分钟内响应时间大，是因为每个表初始化完都是一个 region,大量 insert 进来后需要进行分裂，消耗时间比较大。

*   **Raftstore cpu 高问题**：由于 Raftstore 还是单线程，测试中从监控指标看到 CPU 达到瓶颈是raftrestore 线程。

*   **TiKV 性能中的“木桶原理”**：TiKV 中一个节点的写入性能变慢会影响到整个集群的 TPS 与响应时间。

上线时我们做了以下两方面改善：

**1\. 优化表的定义与索引**

表定义：不使用自增长列（自增长的 rowid）作为主键，避免大量 INSERT 时把数据集中写入单个 Region，造成写入热点。

索引：使用有实际含义的列作为主键，同时减少表不必要的索引，以加快写入的速度。

**2\. 对表的 region 进行强制分裂**

查找表对应的 region：`curl http://$tidb_ip:$status_port /tables/$schema/$table_name/regions`

使用 pd-ctl 工具 split 对应表的 region：`operator add split-region $region_id`

打散表的隐式 id，打散表的数据分布：`alter table $table_name shard_row_id_bits=6;`

![](media/user-case-pingankeji/22.PNG)

我们使用了 25 台机器，后面还临时准备了 10 台机器去应对高并发的不时之需。

在使用过程中遇到如下问题：

**（1） 2.0.10 版本下 in 不能下推到表过渡问题**

![](media/user-case-pingankeji/23.PNG)

大家看到我们两个相同的表结构，同时写入一些数据，在两个表进行关联的时候，发现过滤条件 t1.id=1 时，上面那个执行计划可以下推到两个表进行过滤，两个表可以完全精准的把数据取出来，但是下面把等号后改成 in 的时候，对 t2 表进行全表扫描，如果 t2 表数据量很大时就会很慢，这是 TiDB 的一个 bug，解决方案就是不要用 in，**在 2.1 版本修复了这个 bug。**

**（2） 2.0.10 下时区设置导致客户端不能连**

![](media/user-case-pingankeji/24.PNG)

我们在跑命令的时候没有问题，并且结果是可以的，但是跑完后就断掉了，从后台看也是有问题的，重启 TiDB 组件也不行，后来找到代码我们发现这是一个 bug。

**原因**：这个 bug 会在你连接时 check 这个时区，导致用户不能连接。

**解决办法**：我们找研发同事重新编译一个 tidb-server 登入服务器，把时区设置为正确的，然后使用最初的 TiDB 组件登录，**2.1 版本后这个 bug 修复。**

**（3） Spring 框架下 TiDB 事务**

![](media/user-case-pingankeji/25.PNG)


这个问题是比较重要的问题，有个产品需要生成一个唯一的保单号，业务是批量生成的，当时在 TiDB 中我们建了一个表，表中只有一条数据，但是我们发现会有重复保单号出来。

**原因**：TiDB 使用乐观事务模型，在高并发执行 Update 语句对同一条记录更新时，不同事务拿的版本值可能是相同的，由于不同事务只有在提交时，才会检查冲突，而不是像 Oracle、MySQL、PG 那样，使用锁机制来实现对一记录的串行化更改。

**解决办法**：Spring 开发框架下，对事务的管理是使用注解式的，无法捕获到 TiDB commit 时的返回状态。因此需要将 spring 注解式事务改成编程式事务，并对 commit 状态进行捕获，根据状态来决定是重试机制，具体步骤：

*   利用 redis 实现分布式锁，执行 SQL。

*   捕获事务 commit 状态，并判断更新成功还是失败：

    *   失败：影响行数为 0 || 影响行数为 1 && commit 时出现异常。

    *   成功：影响行数为 1 && commit 时无异常。


![](media/user-case-pingankeji/26.PNG)


