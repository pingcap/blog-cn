---
title: TiDB at 丰巢：尝鲜分布式数据库
author: ['丰巢技术团队']
date: 2018-11-09
summary: TiDB 的改造完成之后，丰巢推送服务对大部分消息进行了落地和查询，截止目前为止，推送服务最大的日落地量已经达到了 5 千万。
tags: ['互联网']
category: case
url: /cases-cn/user-case-fengchao/
weight: 5
logo: /images/blog-cn/customers/fengchao-logo.png
---


随着丰巢业务系统快速增长，其核心系统的数据量，早就跨越了亿级别，而且每年增量仍然在飞速发展。整个核心系统随着数据量的压力增长，不但系统架构复杂度急剧增长，数据架构更加复杂，传统的单节点数据库，已经日渐不能满足丰巢的需求，当单表数量上亿的时候，Oracle 还能勉强抗住，而 MySQL 到单表千万级别的时候就难以支撑，需要进行分表分库。为此，一款高性能的分布式数据库，日渐成为刚需。

## 思考

在互联网公司业务量增大之后，并行扩展是最常用、最简单、最实时的手段。例如负载均衡设备拆流量，让海量流量变成每个机器可以承受的少量流量，并且通过集群等方式支撑起来整个业务。于是当数据库扛不住的时候也进行拆分。

但有状态数据和无状态数据不同，当数据进行拆分的时候，会发生数据分区，而整个系统又要高可用状态下进行，于是数据的一致性变成了牺牲品，大量的核对工具在系统之间跑着保证着最终的一致性。在业务上，可能业务同学经常会遇到分过库的同学说，这个需求做不了，那个需求做不了，如果有 sql 经验的业务同学可能会有疑问不就是一条 sql 的事情么，其实这就是分库分表后遗症。

为此，我们需要有个数据库帮我们解决以上问题，它的特性应该是：

* 数据强一致：支持完整的 ACID；

* 不分表分库：无论多少数据我们只管插入不需要关心啥时候扩容，会不会有瓶颈；

* 数据高可用：当我们某台数据库的少部分机器磁盘或者其他挂了的时候，我们业务上可以无感知，甚至某个城市机房发生灾难的时候还可以持续提供服务，数据不丢失；

* 复杂 SQL 功能：基本上单库的 SQL，都可以在这个数据库上运行，不需要修改或者些许修改；

* 高性能：在满足高 QPS 的同时，保证比较低的延时。

## 选型

根据以上期望进行分析，我们分析了目前市面上存在的 NewSQL 分布式数据库，列表如下： 

![](https://upload-images.jianshu.io/upload_images/542677-5a820d66fe6d1a99.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**在综合考虑了开源协议，成熟度，可控度，性能，服务支撑等综合因素之后，我们选择了 TiDB，它主要优势如下：**

* 高度兼容 MySQL

  大多数情况下，无需修改代码即可从 MySQL 轻松迁移至 TiDB，分库分表后的 MySQL 集群亦可通过 TiDB 工具进行实时迁移。    

* 水平弹性扩展

  通过简单地增加新节点即可实现 TiDB 的水平扩展，按需扩展吞吐或存储，轻松松应对高并发、海量数据场景。

* 分布式事务 

  TiDB 100% 支持标准的 ACID 事务。

* 金融级别的高可用性

  相比于传统主从（M-S）复制方案，基于 Raft 的多数派选举协议可以提供金融级的 100% 数据强一致性保证，且在不丢失大多数副本的前提下，可以实现故障的自动恢复（auto-failover），无需人工介入。

基于如上的原因，我们选择了 TiDB，作为丰巢的核心系统的分布式数据库，来取代   Oracle 和 MySQL。

## 评估

### 1. 性能测试

TiDB 的基准测试，使用的工具是 sysbanch 进行测试，使用了 8 张基础数据为一千万的表，分别测试了 insert，select，oltp 和 delete 脚本得到数据如下，查询的 QPS 达到了惊人的 14 万每秒，而插入也稳定在 1 万 4 每秒。

核心服务器配置：

![](https://upload-images.jianshu.io/upload_images/542677-995e5b7f363a02df.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

测试结果：

![](https://upload-images.jianshu.io/upload_images/542677-07def9ded73b6070.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过～

### 2. 功能测试


![](https://upload-images.jianshu.io/upload_images/542677-f7ee13595c0635dd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


通过～

## 接入

因为是核心系统，安全起见，我们采取了多种方案保证验证项目接入的可靠性，保证不影响业务。

### 1. 项目选择

在寻找第一个接入项目的时候，我们以下面 4 个特征，进行了选择：

![](https://upload-images.jianshu.io/upload_images/542677-1e6b3da457be7626.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终，我们选择了推送服务。因为推送服务是丰巢用来发送取件通知的核心服务，量非常大，但逻辑简单，而且有备选外部推送方案，所以即便万一出现问题，而不会影响用户。

### 2. 代码修改

因为 TiDB 是完全兼容 MySQL 语法的，所以在这个项目的接入过程中，我们对代码的修改是很细微的。SQL 基本零改动，主要是外围代码，包括：

* 异步接口修改，数据异步化入库

* 同步接口修改，实现异常熔断

* 停止内嵌数据迁移代码

以上三点，保证了整个系统在不强依赖于数据库，并且能在高并发的情况下通过异步落库保护数据库不被压垮，并且在数据库发生问题的时候，核心业务可以正常进行下去。

## 效果

### 1. 查询能力

接入 TiDB 之后，原先按照时间维度来拆分的十几个分表，变成了一张大表。最明显的变化，是在大数据量下，数据查询能力有了显著的提升。 

![](https://upload-images.jianshu.io/upload_images/542677-eef2bf7900e8671d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2. 监控能力

TiDB 拥有很完善的监控平台，可以直观的看到容量，以及节点状态：

![](https://upload-images.jianshu.io/upload_images/542677-9f06fd3b88effdc0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还能了解每个节点负载和 sql 执行的延时：

![](https://upload-images.jianshu.io/upload_images/542677-4912a4c9f12da3a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然还能了解所在机器上的位置，CPU 内存等负载情况：

![](https://upload-images.jianshu.io/upload_images/542677-21130af5ec82e33b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

网络状态也能清晰的监控到：

![](https://upload-images.jianshu.io/upload_images/542677-6de5bfaa854bf2f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所有这些能让团队能分析出来有问题的 sql，以及数据库本身的问题。

## 小结

TiDB 的接入过程，整体还是非常顺利的，由于之前做了很多接入的保障工作，当天切换流量到 TiDB 的过程只用了 10 分钟的时间，在此也要感谢 TiDB 对于 MySQL 语法的兼容性的支持，以及 PingCAP 提供的各种有用的工具。到目前为止，系统的稳定运行了一个多月，很好的满足了丰巢的业务需求。

TiDB 的改造完成之后，丰巢推送服务对大部分消息进行了落地和查询，截止目前为止，推送服务最大的日落地量已经达到了 5 千万，而如果现在推送服务还使用的还是 MySQL 的方案，就需要上各种的分库分表方案，很多细致的业务就无法或者难以开展。

此次 TiDB 的改造，只是丰巢对于分布式数据技术探索的一小步，未来丰巢会将更多的分布式技术，引入到更多的业务系统，打造更加极致的产品和服务。 