---
title: TiDB / TiSpark 在易果集团实时数仓中的创新实践
author: ['罗瑞星']
date: 2017-12-18
summary: 同时解决 OLAP 和 OLTP 是一件相当困难的事情，TiDB 和 TiSpark 虽然推出不久，但是已经满足很多应用场景，同时在易用性和技术支持上也非常值得称赞。
tags: ['互联网']
category: case
url: /cases-cn/user-case-yiguo/
aliases: ['/blog-cn/user-case-yiguo/']
weight: 7
logo: /images/blog-cn/customers/yiguo-logo.png
---

## 项目背景  

目前企业大多数的数据分析场景的解决方案底层都是围绕 Hadoop 大数据生态展开的，常见的如 HDFS + Hive + Spark + Presto + Kylin，在易果集团，我们初期也是采取这种思路，但是随着业务规模的快速增长和需求的不断变化，一些实时或者准实时的需求变得越来越多，这类业务除了有实时的 OLTP 需求，还伴随着一些有一定复杂度的 OLAP 的需求，单纯地使用 Hadoop 已经无法满足需求。

现有的准实时系统运行在 SQL Server 之上，通过开发人员编写和维护相应的存储过程来实现。由于数据量不大，SQL Server  能够满足需求，但是随着业务的发展，数据量随之增长，SQL Server  越来越不能满足需求，当数据量到达一定的阶段，性能便会出现拐点。这个时候，这套方案已完全无法支撑业务，不得不重新设计新的方案。


## 选型评估

在评估初期，Greenplum、Kudu、TiDB 都进入了我们的视野，对于新的实时系统，我们有主要考虑点：

+ 首先，系统既要满足 OLAP 还要满足 OLTP 的基本需求；

+ 其次，新系统要尽量降低业务的使用要求；

+ 最后，新系统最好能够与现有的 Hadoop 体系相结合。

Greenplum 是一套基于 PostgreSQL 分析为主的 MPP 引擎，大多用在并发度不高的离线分析场景，但在 OLTP 方面，我们的初步测试发现其对比 TiDB 的性能差很多。

再说说 Kudu。Kudu 是 CDH 2015年发布的一套介于 Hbase 和 HDFS 中间的一套存储系统，目前在国内主要是小米公司应用的较多，在测试中，我们发现其在 OLTP 表现大致与 TiDB 相当，但是一些中等数据量下，其分析性能相比 TiDB 有一定差距。另外我们的查询目前主要以 Presto 为主，Presto 对接 Kudu 和 PostgreSQL 都是需要考虑兼容性的问题，而 TiDB 兼容 MySQL 协议，在应用初期可以直接使用 Presto-MySQL 进行统一查询，下一步再考虑专门开发 Presto-TiDB。

另外，我们希望未来的实时系统和离线系统能够通用，一套代码在两个系统中都能够完全兼容，目前 Tispark 和 SparkSQL 已经很大程度上实现了这点，这支持我们在以后离线上的小时级任务可以直接切换到 TiDB上，在 TiDB 上实现实时业务的同时，如果有 T+1 的需求也能够直接指 HDFS 即可，不用二次开发，这是 Kudu 和 GP 暂时实现不了的。     

最后，TiSpark 是建立在 Spark 引擎之上，Spark 在机器学习领域里有诸如 Mllib 等诸多成熟的项目，对比 GP 和 Kudu，算法工程师们使用 TiSpark 去操作 TiDB 的门槛非常低，同时也会大大提升算法工程师们的效率。

经过综合的考虑，我们最终决定使用 TiDB 作为新的实时系统。同时，目前 TiDB 的社区活跃度非常好，这也是我们考虑的一个很重要的方面。

## TiDB 简介

在这里介绍一下 TiDB 的相关特性：TiDB 是基于 Google Spanner/F1 论文启发开源的一套 [NewSQL 数据库](https://github.com/pingcap/tidb)，它具备如下 NewSQL 核心特性：

+ SQL支持 （TiDB 是 MySQL 兼容的）

+ 水平线性弹性扩展

+ 分布式事务

+ 数据强一致性保证

+ 故障自恢复的高可用

同时，TiDB 还有一套丰富的生态工具，例如：快速部署的 TiDB-Ansible、无缝迁移 MySQL 的 Syncer、异构数据迁移工具 Wormhole、以及 TiDB-Binlog、Backup & Recovery 等。

## SQL Server 迁移到 TiDB

由于我们公司的架构是 .NET + SQL Server 架构，所以我们无法像大多数公司一样去使用 MySQL Binlog 去做数据同步，当然也就无法使用 TiDB 官方提供的 Syncer 工具了。因此我们采用了 Flume + Kafka 的架构，我们自己开发了基于 Flume 的 SQL Server Source 去实时监控 SQL Server 数据变化，进行捕捉并写入 Kafka 中，同时，我们使用 Spark Streaming 去读取 Kafka 中的数据并写入 TiDB，同时我们将之前 SQL Server 的存储过程改造成定时调度的 MySQL 脚本。

![图：SQL Server 数据迁移到 TiDB](http://upload-images.jianshu.io/upload_images/542677-cc08a4bab6425e1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## TiDB 前期测试

在测试初期，我们采用 TiDB 的版本为 RC4，在测试过程中曾经在同时对一张表进行读写时，出现 Region is stale 的错误，在 GitHub 上提出 Issue 后，TiDB 官方很快在 Pre-GA 版本中进行了修复。在测试环境，我们是手动通过二进制包的形式来部署 TiDB ，虽然比较简单，但是当 TiDB 发布 GA 版本之后，版本升级却是一个比较大的问题，由于早期没有使用 TiDB-ansible 安装，官方制作的升级脚本无法使用，而手动进行滚动升级等操作非常麻烦。

由于当时是测试环境，在听取了 TiDB 官方的建议之后，我们重新利用 TiDB 官方提供的 TiDB-ansible 部署了 TiDB 的 GA 版本。只需要下载官方提供的包，修改相应的配置，就能完成安装和部署。官方也提供了升级脚本，能够在相邻的 TiDB 版本之前完成无缝滚动升级。同时 TiDB-ansible 默认会提供 Prometheus + Grafana 的监控安装，官方提供了非常丰富完善的 Grafana 模板，省去了运维很多监控配置的工作量，借着 TiDB 部署监控的契机，我们也完成了诸如 Redis，RabbitMQ，Elasticsearch 等很多应用程序的监控由 Zabbix 往 Prometheus 的迁移。这里需要注意的是，如果是用官方提供的部署工具部署 Prometheus 和 Grafana，在执行官方的停止脚本时切记跳过相应的组件，以免干扰其他程序的监控。

## TiDB 上线过程

在 10 月中旬，随着新机器的采购到位，我们正式将 TiDB 部署到生产环境进行测试，整个架构为 3 台机器，3TiKV＋3PD＋2TiDB 的架构。在生产环境中的大数据量场景下，遇到了一些新的问题。

首先遇到的问题是 OLTP 方面，Spark Streaming 程序设置的 5 秒一个窗口，当 5 秒之内不能处理完当前批次的数据，就会产生延迟，同时 Streaming 在这个批次结束后会马上启动下一个批次，但是随着时间的积累，延迟的数据就会越来越多，最后甚至延迟了 8 小时之久；另一方面，由于我们使用的是机械硬盘，因此写入的效率十分不稳定，这也是造成写入延迟的一个很主要的因素。

出现问题之后我们立即与 TiDB 官方取得联系，确认 TiDB 整体架构主要基于 SSD 存储性能之上进行设计的。我们将 3 台机器的硬盘都换成了 SSD；与此同时，我们的工程师也开发了相应的同步程序来替代 Spark Streaming，随着硬件的更新以及程序的替换，写入方面逐渐稳定，程序运行的方式也和 Streaming 程序类似，多程序同时指定一个 Kafka 的 Group ID，同时连接不同机器的 TiDB 以达到写入效率最大化，同时也实现了 HA，保证了即使一个进程挂掉也不影响整体数据的写入。

在 OLTP 优化结束之后，随之而来的是分析方面的需求。由于我们对 TiDB 的定位是实时数据仓库，这样就会像 Hadoop 一样存在很多 ETL 的流程，在 Hadoop 的流程中，以 T+1 为主的任务占据了绝大多数，而这些任务普遍在凌晨启动执行，因此只能用于对时间延迟比较大的场景，对实时性要求比较高的场景则不适合，而 TiDB 则能很好的满足实时或者准实时的需求，在我们的业务场景下，很多任务以 5-10 分钟为执行周期，因此，必须确保任务的执行时长在间隔周期内完成。

我们取了两个在 SQL Server 上跑的比较慢的重要脚本做了迁移，相比于 SQL Server／MySQL 迁移至 Hadoop，从 SQL Server 迁移至 TiDB 的改动非常小，SQL Server 的 Merge 操作在 TiDB 里也通过 replace into 能够完成，其余一些 SQL Server 的特性，也能够通过 TiDB 的多行事务得以实现，在这一方面，TiDB 的 GA 版本已经做的非常完善，高度兼容 MySQL，因此迁移的成本非常小，从而使我们能够将大部分精力放在了调优方面。

在脚本迁移完毕之后，一些简单的脚本能够在秒级完成达到了我们的预期。但是一些复杂的脚本的表现在初期并没表现出优势，一些脚本与 SQL Server 持平甚至更慢，其中最大的脚本 SQL 代码量一共 1000 多行，涉及将近 20 张中间表。在之前的 SQL Server 上，随着数据量慢慢增大，每天的执行时长逐渐由 1-2 分钟增长到 5-6 分钟甚至更久，在双11当天凌晨，随着单量的涌入和其他任务的干扰延迟到 20 分钟甚至以上。在迁移至 TiDB 初期，在半天的数据量下 TiDB 的执行时长大致为 15 分钟左右，与 SQL Server 大致相同，但是并不能满足我们的预期。我们参考了 TiDB 的相关文档对查询参数做了一些调优，几个重要参数为：tidb_distsql_scan_concurrency，tidb_index_serial_scan_concurrency，tidb_index_join_batch_size（TiDB 提供了很好的并行计算能力）。经过验证，调整参数后，一些 SQL 能够缩短一倍的执行时间，但这里依旧不能完全满足我们的需求。

## 引入 TiSpark

随后，我们把目光转向了 TiDB 的一个子项目 [TiSpark](https://github.com/pingcap/tispark)，用官网的介绍来讲 TiSpark 就是借助 Spark 平台，同时融合 TiKV 分布式集群的优势，和 TiDB 一起解决 HTAP 的需求。TiDB-ansible 中也带有 TiSpark 的配置，由于我们已经拥有了 Spark 集群，所以直接在现有的 Spark 集群中集成了 TiSpark。虽然该项目开发不久，但是经过测试，收益非常明显。

TiSpark 的配置非常简单，只需要把 TiSprak 相关的 jar 包放入 Spark 集群中的 jars 文件夹中就能引入 TiSpark，同时官方也提供了 3 个脚本，其中两个是启动和停止 TiSpark 的 Thrift Server，另一个是提供的 TiSpark 的 cli 客户端，这样我们就能像使用 Hive 一样使用 TiSpark 去做查询。

在初步使用之后，我们发现一些诸如 select count(*) from table 等 SQL 相比于 TiDB 有非常明显的提升，一些简单的 OLAP 的查询基本上都能够在 5 秒之内返回结果。经过初步测试，大致在 OLAP 的结论如下：一些简单的查询 SQL，在数据量百万级左右，TiDB 的执行效率可能会比 TiSpark 更好，在数据量增多之后 TiSpark 的执行效率会超过 TiDB，当然这也看 TiKV 的配置、表结构等。在 TiSpark 的使用过程中，我们发现 TiSpark 的查询结果在百万级时，执行时间都非常稳定，而 TiDB 的查询时间则会随着数据量的增长而增长（经过与 TiDB 官方沟通，这个情况主要是因为没有比较好的索引进行数据筛选）。针对我们的订单表做测试，在数据量为近百万级时，TiDB 的执行时间为 2 秒左右，TiSpark 的执行时间为 7 秒；当数据量增长为近千万级时，TiDB 的执行时间大致为 12 秒（不考虑缓存），TiSpark 依旧为 7 秒，非常稳定。

因此，我们决定将一些复杂的 ETL 脚本用 TiSpark 来实现，对上述的复杂脚本进行分析后，我们发现，大多数脚本中间表很多，在 SQL Server 中是通过 SQL Server 内存表实现，而迁移至 TiDB，每张中间表都要删除和插入落地，这些开销大大增加了执行时长（据官方答复 TiDB 很快也会支持 View、内存表）。在有了 TiSpark 之后，我们便利用 TiSpark 将中间表缓存为 Spark 的内存表，只需要将最后的数据落地回 TiDB，再执行 Merge 操作即可，这样省掉了很多中间数据的落地，大大节省了很多脚本执行的时间。

在查询速度解决之后，我们发现脚本中会有很多针对中间表 update 和 delete 的语句。目前 TiSpark 暂时不支持 update 和 delete 的操作（和 TiSpark 作者沟通，后续会考虑支持这两个操作），我们便尝试了两种方案，一部分执行类似于 Hive，采用 insert into 一张新表的方式来解决；另外一部分，我们引入了 Spark 中的 Snappydata 作为一部分内存表存储，在 Snappydata 中进行 update 和 delete，以达到想要的目的。因为都是 Spark 的项目，因此在融合两个项目的时候还是比较轻松的。

最后，关于实时的调度工具，目前我们是和离线调度一起进行调度，这也带来了一些问题，每次脚本都会初始化一些 Spark 参数等，这也相当耗时。在未来，我们打算采用 Spark Streaming 作为调度工具，每次执行完成之后记录时间戳，Spark Streaming 只需监控时间戳变化即可，能够避免多次初始化的耗时，通过 Spark 监控，我们也能够清楚的看到任务的延迟和一些状态，这一部分将在未来进行测试。

## TiDB 官方支持

在迁移过程中，我们得到了 TiDB 官方很好的支持，其中也包括 TiSpark 相关的技术负责人，一些 TiSpark 的 Corner Case 及使用问题，我们都会在群里抛出，TiDB 的官方人员会非常及时的帮助我们解决问题，在官方支持下，我们迁移至 TiSpark 的过程很顺利，没有受到什么太大的技术阻碍。

## 实时数仓 TiDB / TiSpark

在迁移完成之后，其中一条复杂的 SQL，一共 Join 了 12 张表（最大表数量亿级，部分表百万级），在平时小批量的情况下，执行时间会在 5 分钟左右，我们也拿了双 11 全量的数据进行了测试，执行时间在 9 分钟以上，而采用了 TiSpark 的方式去执行，双 11 全量的数据也仅仅花了 1 分钟，性能提升了 9 倍。整个大脚本在 SQL Server 上运行双 11 的全量数据以前至少要消耗 30 分钟，利用 TiDB 去执行大致需要 20 分钟左右，利用 TiSpark 只需要 8 分钟左右，相对 SQL Server 性能提升 4 倍，也就是说，每年数据量最高峰的处理能力达到了分钟级，很好的满足了我们的需求。

最后，不管是用 TiDB 还是用 TiSpark 都会有一部分中间表以及与原表进行 Merge 的操作，这里由于 TiDB 对事务进行的限制，我们也采用以万条为单批次进行批量的插入和 Merge，既避免了超过事务的报错又符合 TiDB 的设计理念，能够达到最佳实践。

有了 TiSpark 这个项目，TiDB 与 Hadoop 的生态体系得到进一步的融合，在没有 TiSpark 之前，我们的系统设计如下：

![图：多套数仓并存](http://upload-images.jianshu.io/upload_images/542677-7e8a2d166704cfbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以发现，实时数仓与 T+1 异步数仓是两个相对独立的系统，并没有任何交集，我们需要进行数据实时的同步，同时也会在夜晚做一次异步同步，不管是 Datax 还是 Sqoop 读取关系型数据库的效率都远远达不到 TiSpark 的速度，而在有了 TiSpark 之后，我们可以对 T+1 异步数仓进行整合，于是我们的架构进化为如下：

![图：TiDB / TiSpark 实时数仓平台](http://upload-images.jianshu.io/upload_images/542677-03f1ffb1adc9bb6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样就能够利用 TiSpark 将 TiDB 和 Hadoop 很好的串联起来，互为补充，TiDB 的功能也由单纯的实时数仓变成能够提供如下几个功能混合数据库：

1.   实时数仓，上游 OLTP 的数据通过 TiDB 实时写入，下游 OLAP 的业务通过 TiDB / TiSpark 实时分析。

2.   T+1 的抽取能够从 TiDB 中利用 TiSpark 进行抽取。

    + TiSpark 速度远远超过 Datax 和 Sqoop 读取关系型数据库的速度；

    + 抽取工具也不用维护多个系统库，只需要维护一个 TiDB 即可，大大方便了业务的统一使用，还节省了多次维护成本。

    + TiDB 天然分布式的设计也保证了系统的稳定、高可用。

3.   TiDB 分布式特性可以很好的平衡热点数据，可以用它作为业务库热点数据的一个备份库，或者直接迁入 TiDB 。

上面这三点也是我们今后去努力的方向，由此可见，TiSpark 不仅对于 ETL 脚本起到了很重要的作用，在我们今后的架构中也起到了举足轻重的作用，为我们创建一个实时的统一的混合数据库提供了可能。

与此同时，我们也得到 TiDB 官方人员的确认，TiDB 将于近期支持视图、分区表，并会持续增强 SQL 优化器，同时也会提供一款名为 TiDB Wormhole 的异构平台数据实时迁移工具来便捷的支持用户的多元化迁移需求。我们也计划将更多的产品线逐步迁入 TiDB。

## 总结

同时解决 OLAP 和 OLTP 是一件相当困难的事情，TiDB 和 TiSpark 虽然推出不久，但是已经满足很多应用场景，同时在易用性和技术支持上也非常值得称赞，相信 TiDB 一定能够在越来越多的企业中得到广泛应用。


> 作者简介：罗瑞星，曾就职于前程无忧，参加过 Elasticsearch 官方文档中文翻译工作，现就职于易果集团，担任资深大数据工程师，负责易果集团数据分析架构设计等工作。
