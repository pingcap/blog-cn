---
title: TiDB 助力客如云餐饮 SaaS 服务
author: ['客如云 BigData Infra Team']
date: 2018-05-25
summary: 目前 TiDB 线上已经存储超过 6 个月的数据，总数据量几 T，支持线上的查询和分析需求，很多一般复杂度 OLAP 查询都能够在秒级返回结果。
tags: ['互联网']
category: case
url: /cases-cn/user-case-keruyun/
weight: 15
logo: /images/blog-cn/customers/keruyun-logo.png
---


## 公司介绍

客如云成立于 2012 年，是全球领先、 国内最大的 SaaS 系统公司。 目前面向餐饮、 零售等服务业商家， 提供软硬一体的新一代智能化前台、收银等 SaaS 云服务，包括预订、排队、外卖、点餐、收银、会员管理、进销存等系统服务，并将数据实时传达云端。我们是客如云的大数据基础架构组，负责公司的大数据架构和建设工作，为公司提供大数据基础数据服务。

## 业务发展遇到的痛点

1\. 随着公司业务架构越来越复杂，技术架构组需要在服务器端与应用端尽可能的通过微服务化实现业务解耦，同时需要第三方熔断服务机制来保证核心业务正常运行。数据库层面，为了保证高并发的实时写入、实时查询、实时统计分析，我们针对地做了很多工作，比如对实时要求较高的服务走缓存、对大表进行分库分表操作、对有冷热属性的大表进行归档、库分离，虽然用大量人力资源解决了部分问题，但是同时也带来了历史数据访问、跨库分表操作、多维度查询等问题。

2\. 大数据量下，MySQL 稍微复杂的查询都会很慢，线上业务也存在单一复杂接口包含执行几十次 SQL 的情况，部分核心交易大库急需解决访问性能。

3\. 餐饮行业有明显的业务访问高峰时间，高峰期期间数据库会出现高并发访问，而有些业务，比如收银，在高峰期出现任何 RDS 抖动都会严重影响业务和用户体验。

4\. 传统的数仓业务往往有复杂的 T+1 的 ETL 过程，无法实时的对业务变化进行动态分析和及时决策。

## 业务描述

大数据的 ODS（Operational Data Store）以前选型的是 MongoDB，ODS 与支持 SaaS 服务的 RDS 进行数据同步。初期的设想是线上的复杂 SQL、分析 SQL，非核心业务 SQL 迁移到大数据的 ODS 层。同时 ODS 也作为大数据的数据源，可以进行增量和全量的数据处理需求。但是由于使用的 MongoDB，对业务有一定侵入，需要业务线修改相应查询语句，而这点基本上遭到业务线的同学不同程度的抵制。同时目前大数据使用的是 Hadoop + Hive 存储和访问方案，业务线需要把历史明细查询迁移到 Hadoop ，然后通过 Impala、Spark SQL、Hive SQL 进行查询，而这三个产品在并发度稍微高的情况下，响应时间都会很慢，所以大数据组在提供明细查询上就比较麻烦。 

同时更为棘手的是，面对客户查询服务（历史细则、报表等），这类查询的并发会更高，而且客户对响应时间也更为敏感，我们首先将处理后的数据（历史细则等）放在了 MongoDB 上（当时 TiDB 1.0 还没有 GA，不然就使用 TiDB 了），然后针对 OLAP 查询使用了 Kylin ，先解决了明细查询的问题。 但是由于业务很复杂, 数据变更非常频繁，一条数据最少也会经过五六次变更操作。报表展现的不仅是当天数据，涉及到挂账、跨天营业、不结账、预定等复杂状况，生产数据的生命周期往往会超过一个月以上。所以当前的 OLAP 解决方案还有痛点，所以后续我们要把 OLAP 查询移植一部分到 TiDB 上面去，来减轻 Kylin 的压力并且支持更加灵活的查询需求，这个目前还在论证当中。 

同时，我们发现 TiDB 有一个子项目 TiSpark， TiSpark 是建立在 Spark 引擎之上，Spark 在机器学习领域里有诸如 MLlib 等诸多成熟的项目，算法工程师们使用 TiSpark 去操作 TiDB 的门槛非常低，同时也会大大提升算法工程师们的效率。我们可以使用 TiSpark 做 ETL，这样我们就能做到批处理和实时数仓，再结合 CarbonData 可以做到非常灵活的业务分析和数据支持，后期根据情况来决定是否可以把一部分 Hive 的数据放在 TiDB 上。

新老框架如下图：

![图：老的框架](http://upload-images.jianshu.io/upload_images/542677-a1df896622f704fb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图：老的框架</center>

![图：新的框架](http://upload-images.jianshu.io/upload_images/542677-c458370c2f45d4a0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<center>图：新的框架</center>

## TiDB 测试应用

### 1. 配置

阿里云服务器：

*   TiDB / PD：3 台 i1 型 机器，16c 64g ；

*   TiKV ：5 台 i2 型机器，16c 128g， 1.8T*2 每台机器部署两个 KV；

*   监控机一台。

目前我们将线上 RDS 中三个库的数据通过 Binlog 同步到 TiDB ，高峰期 QPS 23k 左右，接入了业务端部分查询服务；未来我们会将更多 RDS 库数据同步过来，并交付给更多业务组使用。因为 TiDB 是新上项目，之前的业务线也没有线上 SQL 迁移的经历，所以在写入性能上也没有历史数据对比。

![](http://upload-images.jianshu.io/upload_images/542677-bd84a89f3f5a968b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2. 性能对比

（1）查询一个索引后的数字列，返回 10 条记录，测试索引查询的性能。

![](http://upload-images.jianshu.io/upload_images/542677-e9b8c6ab8727d3b9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（2）查询两个索引后的数字列，返回 10 条记录（每条记录只返回 10 字节左右的 2 个小字段）的性能，这个测的是返回小数据量以及多一个查询条件对性能的影响。

![](http://upload-images.jianshu.io/upload_images/542677-a8cd97f157086ca1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（3）查询一个索引后的数字列，按照另一个索引的日期字段排序（索引建立的时候是倒序，排序也是倒序），并且 Skip 100 条记录后返回 10 条记录的性能，这个测的是 Skip 和 Order 对性能的影响。

![](http://upload-images.jianshu.io/upload_images/542677-a071a948645b68ab?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（4）查询 100 条记录的性能（没有排序，没有条件），这个测的是大数据量的查询结果对性能的影响。

![](http://upload-images.jianshu.io/upload_images/542677-5c2116de0650aad5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（5）TiDB 对比 MySQL 复杂 SQL 执行速率：

*   Table 1  TiDB 数据量 5 千万，MySQL数据量 2.5 千万；

*   Table 2  TiDB 数据量 5 千万，MySQL数据量 2.5 千万；

*   Table 3  TiDB 数据量 5 千万，MySQL数据量 2.5 千万。

**a. 对应 SQL：**

```

SELECT sum(p.exempt_amount) exempt_amount FROM table1 p JOIN table2 c ONp.relate_id=c.id  AND p.is_paid = 1

andp.shop_identy in(BBBBB)  

andp.brand_identy=AAAAA

andp.is_paid=1 AND p.status_flag=1 AND p.payment_type!=8              

WHEREc.brand_identy = AAAAA

ANDc.shop_identy in(BBBBB)                              

ANDc.trade_type in(1,3,4,2,5)                          

ANDc.recycle_status=1        

AND c.trade_statusIN (4,5,10)        

ANDp.payment_time BETWEEN '2017-08-11 16:56:19' AND '2018-01-13 00:00:22'        

ANDc.status_flag = 1        

ANDc.trade_pay_status in(3,5)                    

AND c.delivery_type in(1,2,3,4,15)
```

![](http://upload-images.jianshu.io/upload_images/542677-b91ac6ac0408c699?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 **b. 对应 SQL：**

```

SELECT sum(c.sale_amount)tradeAmount,sum(c.privilege_amount) privilege_amount,sum(c.trade_amount)totalTradeAmount,sum(c.trade_amount_before) tradeAmountBefore        

FROM (SELECTc.sale_amount,c.privilege_amount,c.trade_amount,c.trade_amount_before        

FROM table1p        

JOIN table2c ON p.relate_id=c.id                                

andp.shop_identy in(BBBBB)                  

andp.brand_identy=AAAAA

andp.is_paid=1 AND p.status_flag=1 AND p.payment_type!=8              

and  c.brand_identy = AAAAA

ANDc.shop_identy in(BBBBB)                                

ANDc.trade_type in(1,3,4,2,5)                            

ANDc.recycle_status=1         AND c.trade_statusIN (4,5,10)        

ANDp.payment_time BETWEEN '2017-07-31 17:38:55' AND '2018-01-13 00:00:26'        

ANDc.status_flag = 1        

ANDc.trade_pay_status in(3,5)                      

ANDc.delivery_type in(1,2,3,4,15)                                  

ANDp.payment_type not in(4,5,6,8,9,10,11,12)        

GROUP BY p.relate_id  ) c
```

![](http://upload-images.jianshu.io/upload_images/542677-2c3dbbf4df319fd6?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **c. 对应 SQL：**

```

SELECT SUM(if(pay_mode_id=-5 or pay_mode_id = -6,0,IFNULL(pi.face_amount, 0) - IFNULL(pi.useful_amount, 0) -IFNULL(pi.change_amount, 0))) redundant

FROM table2c

JOIN  table1 p ON c.id = p.relate_id AND c.brand_identy=p.brand_identy        

JOIN table3pi ON pi.payment_id=p.id AND pi.pay_status in (3,5,10)

AND  pi.brand_identy=p.brand_identy ANDpi.pay_mode_id!=-23                                

andp.shop_identy in(BBBBB)                  

andp.brand_identy=AAAAA

andp.is_paid=1 AND p.status_flag=1 AND p.payment_type!=8              

WHEREc.brand_identy = AAAAA

ANDc.shop_identy in(BBBBB)                              

ANDc.trade_type in(1,3,4,2,5)                          

ANDc.recycle_status=1        

AND c.trade_statusIN (4,5,10)        

ANDp.payment_time BETWEEN '2017-07-31 17:38:55' AND '2018-01-13 00:00:26'        

ANDc.status_flag = 1        

ANDc.trade_pay_status in(3,5)                    

AND c.delivery_type in(1,2,3,4,15)
```

![](http://upload-images.jianshu.io/upload_images/542677-4623b5c6a45ed127?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**d. 对应 SQL：**

```

SELECT  t.id tradeId,sum(t.trade_amount - t.trade_amount_before) AS roundAmount,  sum(-p.exempt_amount) AS exemptAmount

FROM table2t

LEFT JOINtable1 p ON p.relate_id = t.id

LEFT JOINtable3 pi ON pi.payment_id = p.id

WHEREt.brand_identy =AAAAA AND t.trade_status IN (4,5,10)

ANDt.trade_pay_status IN (3,4,5,6,8)  ANDp.payment_type IN (1,2)

ANDpi.pay_mode_id !=-23  ANDp.is_paid=1  AND  t.status_flag=1

AND t.shop_identy IN(<123个商户号码>)

GROUP BY t.id
```

![](http://upload-images.jianshu.io/upload_images/542677-2e4687a243c4d125?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**e. 对应 SQL：**

```

SELECT  t.id tradeId,  

sum(t.trade_amount- t.trade_amount_before) AS roundAmount,

sum(-p.exempt_amount)AS exemptAmount

FROM table2t

 JOIN table1 p ON t.id = p.relate_id

WHERE  t.brand_identy = AAAA

ANDt.trade_status IN(4,5,10)

ANDt.trade_pay_status IN (3,4,5,6,8)  

ANDp.is_paid=1  AND  t.status_flag=1

group by t.id ;
```

![](http://upload-images.jianshu.io/upload_images/542677-6bb750b9fc3e5cea?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（6）OLTP 对比测试结果：

![](http://upload-images.jianshu.io/upload_images/542677-c574063f7389fde5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/542677-426667ee175decb0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/542677-172723f1eb8d214f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（7）简单测试结论：

* 不管是索引查询、分页查询、线上业务级的负载查询，大数据量下 TiDB 的性能都比 MySQL 更强；

* TiDB 整体性能表现满足我们业务的需求。

## 生产使用情况

目前线上已经存储超过 6 个月的数据，总数据量几 T，支持线上的查询和分析需求，很多一般复杂度 OLAP 查询都能够在秒级返回结果。TiSpark 我们也调试通过，准备移植一些支持 OLAP 的 ETL 应用做到实时 ETL。目前 TiDB 生产还有很多优化的空间，比如系统参数，SQL 的使用姿势，索引的设计等等。

## 未来规划

* 已经有一个交易量很大业务部门在向我们了解 TiDB，有可能要使用 TiDB 作为线上交易系统；

* 后续大数据也会使用 TiSpark 来做 OLAP 查询和数据处理，同时也可能会作为 Kylin 的数据源；

* 可以预见将来不管是 OLTP，还是 OLAP 场景，TiDB 都会在客如云发挥越来越重要的作用。

## 致谢

感谢 TiDB 的厂商的人员给予了我们巨大的支持，希望我们能够提供给 TiDB 一些有意义的信息和建议，在 TiDB 成长的路上添砖加瓦。


>作者：客如云 BigData Infra Team 