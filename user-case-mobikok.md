---
title: TiDB 在 Mobikok 广告系统中的应用和实践
author: ['rayi']
date: 2018-04-18
summary: TiDB 对于像我们这样可预期核心数据会暴增的场景，有非常大的意义。在后端支撑力量有限时，业务暴增时只需要增加机器，而不是频繁重构业务，让我们有更多精力在自己的业务上耕耘。
tags: ['互联网']
category: case
url: /cases-cn/user-case-mobikok/
weight: 20
logo: /images/blog-cn/customers/mobikok-logo.png
---


## 公司介绍
Mobikok（可可网络）成立于 2013 年，是一家快速成长的移动互联网营销公司，专注于移动 eCPM 营销。总部在中国深圳，聚焦于订阅 offer 的海外流量变现业务。Mobikok 提供的接口方式支持各类手机端流量（API、SDK、Smartlink），RTB（实时竞价系统）对接海外的 DSP（Demand-Side Platform，需求方平台）高效优化客户的广告效果。截止目前，系统已对 2 亿用户进行广告优化，已接入上百家广告主以及上百家渠道，Mobikok 致力于高效，便捷，专业的帮助广告主以及渠道互惠共赢。
 
## 场景介绍：SSP 系统

订阅 SSP（Sell-Side-Platform）平台当前业务主要分为：SDK、Smartlink、Online API 以及 Offline API；在当前 SSP SDK 业务系统当中，累计用户已达到 2 亿，最初使用的是 MySQL 主从分表的方式存储用户数据，随着数据量的增加，MySQL 单机容量以及大数据量查询成为了瓶颈；当单表数据达到 2 千万以上时，单机 MySQL 的查询以及插入已经不能满足业务的需求，当访问量到一定阶段后，系统响应能力在数据库这一块是一个瓶颈。

一次很偶然的机会在 GitHub 上面了解到 TiDB，并且因为现在业务系统当中使用的 Redis 集群是 Codis，已在线上稳定使用两年，听闻 TiDB 创始团队就是之前 Codis 的作者，所以对 TiDB 有了极大的兴趣并且进行测试。通过测试单机 MySQL 和 TiDB 集群，当数据量达到数千万级别的时候发现 TiDB 效率明显高于 MySQL。所以就决定进行 MySQL 到 TiDB 迁移。

迁移后整体架构图：

![](https://upload-images.jianshu.io/upload_images/542677-b8df2c47baab5455.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 
## 引入 TiDB

在选择使用替换 MySQL 方案当中。我们主要考虑几点：

* 支持 MySQL 便捷稳定的迁移，不影响线上业务；

* 高度兼容 MySQL，少改动代码；

* 支持水平弹性部署服务以及在线升级；

* 支持水平扩展业务；

* 成熟的配套监控服务。

TiDB 数据库整体集群配置：2*TiDB、3*TiKV、3*PD。

![](https://upload-images.jianshu.io/upload_images/542677-c2158162bbe79cfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从 12 月初正式上线到目前为止，TiDB 稳定运行四个多月，最高 QPS 达到 2000，平均 QPS 稳定在 500 左右。TiDB 在性能、可用性、稳定性上完全超出了我们的预期，但是由于前期我们对 TiDB 的了解还不深，在此迁移期间碰到的一些兼容性的问题，比如 TiDB 的自增 ID 的机制，排序的时候需要使用字段名等，咨询 TiDB 的工程师都很快的得到了解决，非常感谢 TiDB 团队的支持以及快速响应。

下图是当前集群的 Grafana 展示图：

![](https://upload-images.jianshu.io/upload_images/542677-cc0dd3109183cdd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 后续计划
使用 TiDB 对于像我们这样可预期核心数据会暴增的场景，有非常大的意义。在后端支撑力量有限时，业务暴增时只需要增加机器，而不是频繁重构业务，让我们有更多精力在自己的业务上耕耘，增加我们的行业竞争力。未来我们还有 ADX（Ad Exchang，广告交易平台）和 DSP 业务，需要处理海量的用户数据以及广告数据。目前统计数据这一块当前业务当中使用的是 Spark Streaming，通过和 TiDB 开发团队沟通，官方 TiSpark 可直接引入到当前统计 Spark 群集当中，非常期望在后续开发当中使用 TiSpark。

## 问题建议
在实际应用当中，因为我们切换的并不是只有用户数据表，还迁移了关于广告业务、渠道业务基础数据表。由于 TiDB 是一个分布式数据库，对于一些小表以及 count(*) 操作会影响效率，后来咨询 TiDB 官方得知，TiDB 有不同的隔离级别，SQL 也有高低优先级，如果有全表扫描的需求，可以使用低的隔离级别或者是低的优先级。将来我们就可以直接所有线上业务使用 TiDB 进行替换，最后还是非常感谢 TiDB 团队的支持与帮助。

 
>作者：rayi，深圳可可网络服务端架构负责人
 


