---
title: TiKV 在饿了么的大规模应用实践
author: ['陈东明']
date: 2018-04-02
summary: 目前，TiKV 的应用会影响饿了么全平台 80% 的流量，包括从用户选餐下单到订单配送整个饿了么流程。
tags: ['互联网']
category: case
url: /cases-cn/user-case-eleme-1/
weight: 10
logo: /images/blog-cn/customers/eleme-logo.png
---

## 背景介绍

饿了么从 2008 年创建以来，一直都是飞速的发展。目前，饿了么已覆盖了 2000 多个城市，拥有 2.6 亿的用户，130 万的商户，300万的骑手。饿了么在配送时间上追求卓越，目前饿了么的准时达订单平均配送时长已达到 28 分钟以内。从 2015 年开始，饿了么形成了 2 大业务，在线交易平台业务和即时配送平台业务。饿了么的用户量和订单量的快速增长，带来了数据量的快速增长，从而产生对大数据量存储的强烈需求，并且很多数据都是 KeyValue 格式的数据。之前饿了么没有统一的 Key-Value 存储系统，这部分数据被存储在 MySQL、Redis、Mongo、Cassandra 等不同的系统中，将数据存储在这些系统中，带来一些问题，比如数据扩容不方便、内存不可靠、性能不达标、运维不方便等。

![](http://upload-images.jianshu.io/upload_images/542677-b7598894fea0ae21?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们希望用一套统一的 Key-Value 存储系统来存储这些 Key-Value 形式的数据，并且满足以下所有的技术要求： 

*   大数据量，可以存储至少数十 TB 级别的数据。

*   高性能，在满足高 QPS 的同时，保证比较低的延时。

*   高可靠，数据被可靠的持久化存储，少量机器的损坏不会导致数据的丢失。

*   高可用，作为在线服务的底层依赖存储，要有非常完善的高可用性能力，外卖服务不同于电子商务，对实时性要求非常高，对系统的可用性的要求则是更高的。

*   易运维，可以在不停服的基础上进行数据迁移，和集群扩容。

从 2017 年下半年开始，饿了么开始基于 TiKV 构建饿了么的 Key-Value 存储系统，并且取得了很好的应用效果。饿了么对 Key-Value 系统的使用是在线系统，由离线计算集群生成数据，在线服务消费这些数据。这些在线服务覆盖了饿了么在线交易和即时配送 2 大平台，在线交易平台中的首页搜索、商品品类、商品排序、天降红包等等，即时配送平台中的物流旗手智能调度等，各种在线服务都在使用我们的 Key-Value 系统。 

**目前，TiKV 的应用会影响饿了么全平台 80% 的流量，包括从用户选餐下单到订单配送整个饿了么流程。**

## TiKV集群上线情况

*   目前已在饿了么部署了十几套  TiKV  集群，分别位于北京、上海、广州的四个机房，共计 100+ 机器节点，承载了数十 TB 的数据。

*   配置了完备的监控告警系统，所有集群都已接入，可以在集群出现问题时及时发送告警信息，为集群的正常运行提供了保障。

*   业务高峰期时，最繁忙的一个集群，写入 QPS 近 5w，读取 QPS 近 10w。

*   至今已稳定运行大半年，没有发生过线上事故。 

在 Key-Value 的这个领域中，有着林林总总的开源系统。我们为什么要选择 TiKV 呢？首先要从 TiDB 的架构说起。TiDB 由 TiKV 存储层和 TiDB SQL 层组成。TiKV 层是 TiDB 系统的底层存储层，TiKV 层本身是一个分布式的 Key-Value 存储系统。而 TiDB 层构建于 TiKV 层基础之上，实现了无状态的 SQL 协议层，负责将用户的 SQL 请求，转化为 TiKV 的 Key-Value 请求，从而整体上实现分布式的 SQL 存储。这种架构借鉴了 Google 的 Spanner 系统，Spanner 是一个分布式的 Key-Value 存储系统，Google 在 Spanner 的基础之上，构建了一个名叫 F1 的系统，实现了 SQL 协议。

![](http://upload-images.jianshu.io/upload_images/542677-3b78e7ba3084a852?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们也比较推崇这种架构，并且我们认为在 Key-Value 基础之上，不仅仅可以构建 SQL 协议，也可以构建 Redis 这样的 Key-Value 协议。

![](http://upload-images.jianshu.io/upload_images/542677-55e8b86c5d03206c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这种架构中，上层负责协议转换。TiKV 层则通过数据分片、Raft 协议、MVCC、分布式事务等技术，实现了水平扩展、高可用、强一致性等分布式特性。 

![](http://upload-images.jianshu.io/upload_images/542677-3e488a8b362f7e65?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们的 Redis layer 实现如下：

![](http://upload-images.jianshu.io/upload_images/542677-96720dff45262be7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们构建了一个 ekvproxy 的服务。在这个服务中，我们封装了一个 TiKV 的 SDK，对 Redis 的协议进行了解析，并且将 Redis 协议转成对 TiKV 的调用。并且在这个基础之上，实现了压缩和限流等一些扩展功能。由于我们兼容了 Redis 协议，各种语言均可以在不做任何修改的情况下，直接使用官方的 Redis 客户端访问我们的 Key-Value 服务。在最大程度上减轻了使用方的负担。便于 TiKV 的落地推广。

另外，PingCAP 的工程师还第一时间帮我们实现了 TiKV 的 raw scan 的功能，从而能更好的与 Redis 协议兼容，在此表示感谢。

虽然系统在线上性能表现已经非常不错，但这仍然没有达到 TiKV 的最大处理能力，上线前，我们对 TiKV 进行了详细的性能测试，我们测试环境选在 32 核 CPU，256G 内存，3.5T PCIE SSD 硬盘的机器上， 得到了更好的结果，取其中典型数据如下：

![](http://upload-images.jianshu.io/upload_images/542677-9613f35704f0c7d2?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用 TiKV 的这半年的时间来，在饿了么取得了非常良好的应用效果，应用场景不断增多，数据量不断增大，对饿了么业务做了非常好的支撑，这也依赖于 TiKV 技术人员对我们的各种到位的技术支持。未来，TiKV 在饿了么的应用场景会更加丰富。我们也会考虑将我们 Redis Proxy 开源，开放给社区。

>作者介绍：陈东明，饿了么北京技术中心架构组负责人，负责饿了么的产品线架构设计以及饿了么基础架构研发工作。曾任百度架构师，负责百度即时通讯产品的架构设计。具有丰富的大规模系统构建和基础架构的研发经验，善于复杂业务需求下的大并发、分布式系统设计和持续优化。

