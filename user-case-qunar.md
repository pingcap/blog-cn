---
title: Qunar 高速发展下数据库的创新与发展 - TiDB 篇
author: ['蒲聪']
date: 2017-12-14
summary: 目前已经上线了两个 TiDB 集群。随着产品自身的完善，集群使用量和运维经验的积累，后续我们将逐步推广到更重要的集群中，解决业务上遇到的数据存储的痛点。
tags: ['互联网']
category: case
url: /cases-cn/user-case-qunar/
weight: 12
logo: /images/blog-cn/customers/qunar-logo.png
---


目前互联网公司数据存储主要依赖于 MySQL 为代表的关系型数据库，但是随着业务量的增长，使用场景更加多样，传统的关系型数据库不能很好的满足业务需求，主要是在两个维度：一是随着数据量爆炸式增长，性能急剧下降，而且很难在单机内存储；二是一些场景下业务对响应速度要求较高，数据库无法及时返回结果，而传统的 memcached 缓存又存在无法持久化数据，存储容量受限于内存等问题。针对这两个问题，去哪儿的 DBA 团队分别调研了 TiDB 和 InnoDB memcached 以满足业务需求，为用户提供更多的选择方案。 

接下来的文章系列，我们将陆续为大家带来 TiDB 和 InnoDB memcached 在去哪儿的调研和实践，本篇文章先介绍 TiDB。

## 分布式数据库诞生背景

随着互联网的飞速发展，业务量可能在短短的时间内爆发式地增长，对应的数据量可能快速地从几百 GB 涨到几百个 TB，传统的单机数据库提供的服务，在系统的可扩展性、性价比方面已经不再适用。随着业界相关分布式数据库论文的发布，分布式数据库应运而生，可以预见分布式数据库必将成为海量数据处理技术发展的又一个核心。

目前业界最流行的分布式数据库有两类，一个是以 Google Spanner 为代表，一个是以 AWS Auraro 为代表。 

Spanner 是 shared nothing 的架构，内部维护了自动分片、分布式事务、弹性扩展能力，数据存储还是需要 sharding，plan 计算也需要涉及多台机器，也就涉及了分布式计算和分布式事务。主要产品代表为 TiDB、CockroachDB、OceanBase 等。 

Auraro 主要思想是计算和存储分离架构，使用共享存储技术，这样就提高了容灾和总容量的扩展。但是在协议层，只要是不涉及到存储的部分，本质还是单机实例的 MySQL，不涉及分布式存储和分布式计算，这样就和 MySQL 兼容性非常高。主要产品代表为 PolarDB。

## 去哪儿数据存储方案现状

在去哪儿的 DBA 团队，主要有三种数据存储方案，分别是 MySQL、Redis 和 HBase。

MySQL 是去哪儿的最主要的数据存储方案，绝大部分核心的数据都存储在 MySQL 中。MySQL 的优点不用多说，缺点是没法做到水平扩展。MySQL 要想能做到水平扩展，唯一的方法就业务层的分库分表或者使用中间件等方案。因此几年前就出现了各大公司重复造轮子，不断涌现出中间层分库分表解决方案，比如百度的 DDBS，淘宝的 TDDL，360 的 Atlas 等。但是，这些中间层方案也有很大局限性，执行计划不是最优，分布式事务，跨节点 join，扩容复杂（这个心酸 DBA 应该相当清楚）等。Redis 主要作为缓存使用，针对读访问延时要求高的业务，使用场景有限。  

HBase 因其分布式的特点，可以通过 RS 的增加线性提升系统的吞吐，HDFS 具有水平扩展容量的能力，主要用来进行大数据量的存储，如日志、历史数据、订单快照等。HBase 底层存储是 LSM-Tree 的数据结构，与 B+ Tree 比，LSM-Tree 牺牲了部分读性能，用来大幅提升写性能。 但在实际运维的过程中，HBase 也暴露了一些缺点：  

1\. HBase 能提供很好地写入性能，但读性能就差很多，一是跟本身 LSM-Tree 数据结构有关。二是 HBase 因为要存储大容量，我们采用的是 SAS 盘，用 SSD 成本代价太大。三是跟 HBase 自身架构有关，多次 RPC、JVM GC 和 HDFS 稳定性因素都会影响读取性能。  

2\.  HBase 属于 NoSQL，不支持 SQL，对于使用惯了关系 SQL 的人来说很不方便，另外在运维定位问题的时候也增加了不少难度。比如在运维 MySQL 过程中，可以很快通过慢查询日志就识别出哪些 SQL 执行效率低，快速定位问题 SQL。而在 HBase 运维中，就很难直接识别出客户端怎样的操作很慢，为此我们还在源码中增加了慢查询 Patch，但终归没有直接识别 SQL 来的方便。 

3\. HBase 的软件栈是 Java，JVM 的 GC 是个很头疼的问题，在运维过程中多次出现 RegionServer 因为 GC 挂掉的情况，另外很难通过优化来消除访问延时毛刺，给运维造成了很大的困扰。此外，HBase 在编程语言支持访问对 Java 友好，但是其他语言的访问需要通过 Thrift，有些限制。  

4\. 架构设计上，HBase 本身不存储数据，HBase 通过 RPC 跟底层的 HDFS 进行交互，增加了一次 RPC，多层的架构也多了维护成本。另外 ZooKeeper 的高可用也至关重要，它的不可用也就直接造成了所有 HBase 的不可用。 

5\. HBase 不支持跨行事务。HBase 是 BigTable 的开源实现，BigTable 出现之后，Google 的内部就不停有团队基于 BigTable 做分布式事务，所以诞生了一个产品叫 MegaStore，虽然延时很大，但禁不住好用，所以用的人也很多，BigTable 的作者也后悔当初没有在 BigTable 中加入跨行事务。没有了跨行事务，也就没法实现二级索引，用户只能根据设计 rowkey 索引来进行查询，去哪儿不少业务开发问过我 HBase 是否支持二级索引，结果也不得不因为这个限制放弃了使用 HBase。业界很多公司也在 HBase 上实现二级索引，比如小米、华为等，比较出名的是 Phoenix。我们当初也调研了 Phoenix 来支持二级索引，但在 HBase 本身层次已经很多，Phoenix 无疑又多了一层，在运维的过程中也感觉坑很多 , 此外何时用 Phoenix 何时用 HBase API 的问题也给我们造成很多困扰，最终放弃了这个方案。

综上所述，为了解决现有数据存储方案所遇到的问题，去哪儿 DBA 团队从 2017 年上半年开始调研分布式数据库，在分布式数据库的产品选择上，综合各个方面因素考虑，最终选择了 TiDB，具体原因不在这里细说了，下面开始具体聊聊 TiDB。

## TiDB 架构设计

如下图，TiDB 架构主要分为三个组件：

![image](http://upload-images.jianshu.io/upload_images/542677-ccd260fff7451bdf?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **TiDB Server**：负责接收 SQL 请求，处理 SQL 相关逻辑，通过 PD 找到所需数据的 TiKV 地址。TiDB Server 是无状态的，本身不存储数据，只负责计算，可以无限水平扩展。TiDB 节点可以通过负载均衡组件 (如 LVS、HAProxy 或者 F5) 对外提供统一入口，我个人觉得如果要能实现像 Elasticsearch 那样自己的客户端就更好。

* **PD Server**：Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个:
    * 存储集群元信息（某个 Key 存储在哪个 TiKV 节点）。

    * 对 TiKV 集群进行调度和负载均衡（数据迁移、Raft group leader 迁移等）。
    
    * 分配全局唯一且递增的事务 ID。 

* **TiKV Server**：TiKV 负责存储数据，存储数据基本单位是 Region，每个 TiKV 节点负责管理多个 Region。TiKV 使用 Raft 协议做复制，保证数据一致性和容灾。数据在多个 TiKV 之间的负载均衡由 PD 调度，以 Region 为单位调度。

TiDB 最核心的特性是水平扩展和高可用。
 
* **水平扩展**：TiDB Server 负责处理 SQL 请求，可以通过添加 TiDB Server 节点来扩展计算能力，以提供更高的吞吐。TiKV 负责存储数据，随着数据量的增长，可以部署更多的 TiKV Server 节点来解决数据 Scale 的问题。 

* **高可用**：因 TiDB 是无状态的，可以部署多个 TiDB 节点，前端通过负载均衡组件对外提供服务，这样可以保证单点失效不影响服务。而 PD 和 TiKV 都是通过 Raft 协议来保证数据的一致性和可用性。

## TiDB 原理与实现

TiDB 架构是 SQL 层和 KV 存储层分离，相当于 innodb 插件存储引擎与 MySQL 的关系。从下图可以看出整个系统是高度分层的，最底层选用了当前比较流行的存储引擎 RocksDB，RockDB 性能很好但是是单机的，为了保证高可用所以写多份（一般为 3 份），上层使用 Raft 协议来保证单机失效后数据不丢失不出错。保证有了比较安全的 KV 存储的基础上再去构建多版本，再去构建分布式事务，这样就构成了存储层 TiKV。有了 TiKV，TiDB 层只需要实现 SQL 层，再加上 MySQL 协议的支持，应用程序就能像访问 MySQL 那样去访问 TiDB 了。 

![image](http://upload-images.jianshu.io/upload_images/542677-f3d1ce85c4a52ffa?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里还有个非常重要的概念叫做 Region。MySQL 分库分表是将大的数据分成一张一张小表然后分散在多个集群的多台机器上，以实现水平扩展。同理，分布式数据库为了实现水平扩展，就需要对大的数据集进行分片，一个分片也就成为了一个 Region。数据分片有两个典型的方案：一是按照 Key 来做 Hash，同样 Hash 值的 Key 在同一个 Region 上，二是 Range，某一段连续的 Key 在同一个 Region 上，两种分片各有优劣，TiKV 选择了 Range partition。TiKV 以 Region 作为最小调度单位，分散在各个节点上，实现负载均衡。另外 TiKV 以 Region 为单位做数据复制，也就是一个 Region 保留多个副本，副本之间通过 Raft 来保持数据的一致。每个 Region 的所有副本组成一个 Raft Group, 整个系统可以看到很多这样的 Raft groups。![image](http://upload-images.jianshu.io/upload_images/542677-c53c41f894b1a052?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后简单说一下调度。 TiKV 节点会定期向 PD 汇报节点的整体信息，每个 Region Raft Group 的 Leader 也会定期向 PD 汇报信息，PD 不断的通过这些心跳包收集信息，获得整个集群的详细数据，从而进行调度，实现负载均衡。

## 硬件选型和部署方案

在硬件的选型上，我们主要根据三个组件对硬件不同要求来考虑，具体选型如下：

![image](http://upload-images.jianshu.io/upload_images/542677-7a802e8986ce5774?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因 TiDB 和 PD 对磁盘 IO 要求不高，所以只需要普通磁盘即可。TiKV 对磁盘 IO 要求较高。TiKV 硬盘大小建议不超过 500G，以防止硬盘损害时，数据恢复耗时过长。综合内存和硬盘考虑，我们采用了 4 块 600G 的 SAS 盘，每个 TiKV 机器起 4 个 TiKV 实例，给节点配置 labels 并且通过在 PD 上配置 location-labels 来指明哪些 label 是位置标识，保证同一个机器上的 4 个 TiKV 具有相同的位置标识，同一台机器是多个实例也只会保留一个 Replica。有条件的最好使用 SSD，这样可以提供更好的性能。强烈推荐万兆网卡。 

TiDB 节点和 PD 节点部署在同台服务器上，共 3 台，而 TiKV 节点独立部署在服务器上，最少 3 台，保持 3 副本，根据容量大小进行扩展。 

部署工具使用了 TiDB-Ansible，TiDB-Ansible 是 PingCAP 基于 Ansible playbook 功能编写了一个集群部署工具叫 TiDB-Ansible。使用该工具可以快速部署一个完整的 TiDB 集群（包括 PD、TiDB、TiKV 和集群监控模块)，一键完成以下各项运维工作：

* 初始化操作系统，包括创建部署用户、设置 hostname 等

* 部署组件

* 滚动升级，滚动升级时支持模块存活检测

* 数据清理

* 环境清理

* 配置监控模块

## 监控方案

PingCAP 团队给 TiDB 提供了一整套监控的方案，他们使用开源时序数据库 Prometheus 作为监控和性能指标信息存储方案，使用 Grafana 作为可视化组件进行展示。具体如下图，在 client 端程序中定制需要的 Metric。Push GateWay 来接收 Client Push 上来的数据，统一供 Prometheus 主服务器抓取。AlertManager 用来实现报警机制，使用 Grafana 来进行展示。

![image](http://upload-images.jianshu.io/upload_images/542677-4299162966ff4cb7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下图是某个集群的 Grafana 展示图。

![image](http://upload-images.jianshu.io/upload_images/542677-90d272a7ecf12e0e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## TiDB 使用情况

对于 TiDB 的使用，我们采用大胆实践、逐步推广的思路，一个产品是否经得起考验，需要先用起来，在运维的过程中才能发现问题并不断反馈完善。因此，去哪儿 DBA 经过了充分调研后，8 月下旬开始，我们就先在非核心业务使用，一方面不会因为 TiDB 的问题对现有业务影响过大，另一方面 DBA 自身也积累运维经验，随着产品自身的完善，集群使用量和运维经验的积累，后续我们再逐步推广到更重要的集群中，解决业务上遇到的数据存储的痛点。目前已经上线了两个 TiDB 集群。

* 机票离线集群，主要是为了替换离线 MySQL 库，用于数据统计。当前系统容量是 1.6T，每天以 10G 的增量进行增长，用 MySQL 存储压力较大，且没法扩展，另外一些 OLAP 查询也会影响线上的业务。  

* 金融支付集群，主要是想满足两方面需求，一是当前存储在 MySQL 中的支付信息表和订单信息表都是按月进行分表，运营或者开发人员想对整表进行查询分析，现有的方案是查多个表查出来然后程序进行进一步统计 。二是有些查询并不需要在线上进行查询，只需要是作为线下统计使用，这样就没有必要对线上表建立索引，只需要离线库建立索引即可，这样可以避免降低写性能。  

目前的架构是用 syncer 从线上 MySQL 库做分库分表数据同步到 TiDB 中，然后开发可以在 TiDB 上进行 merge 单表查询、连表 join 或者 OLAP。

>作者：蒲聪，去哪儿平台事业部 DBA，拥有近 6 年的  MySQL 和 HBase 数据库运维管理经验，2014 年 6 月加入去哪儿网，工作期间负责支付平台事业部的 MySQL 和 HBase 整体运维工作，从无到有建立去哪儿网 HBase 运维体系，在 MySQL 和 Hbase 数据库上有丰富的架构、调优和故障处理等经验。目前专注于分布式数据库领域的研究和实践工作。
