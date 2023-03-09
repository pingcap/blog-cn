---
title: 多元生态｜云和恩墨 zCloud 最新支持 TiDB，助力可管理性提升
author: ['PingCAP']
date: 2022-09-30
summary: zCloud 是云和恩墨公司打造的面向多元混合数据库环境，提供跨多云架构、跨多类数据库的一站式智能数据库管理平台。本文将详细介绍 zCloud 中 TiDB 相关管理组件的使用技巧。
tags: ["TiDB"]
---

对于企业级和云数据库，除了性能，可用性和功能等常规维度外，一个重要维度就是可管理性。除了提供必备的「硬」能力以完成用户的技术及业务目标，是否「好用」，是用户做选择时的重要考量，可管理性维度也会很深地影响用户实际使用数据库的隐性成本。TiDB 6.0 版本以来，通过 TiUniManager（原 TiEM）、Clinic 服务等新功能，可管理性大大提升。

与此同时，TiDB 的开源也使得更多生态伙伴加入到了产品体验的优化中。云和恩墨数据库云管平台 zCloud 最新版本实现了对 TiDB 的支持，当前具备集群管理、性能监控、告警和智能巡检等一系列运维管理功能，能够帮助企业用户降低多数据库环境管理复杂度，实现数据库管理的高效与智能。

zCloud 是云和恩墨公司打造的面向多元混合数据库环境，提供跨多云架构、跨多类数据库的一站式智能数据库管理平台。zCloud 以智慧即服务（WaaS - Wisdom as a Service）为产品理念，持续汇聚专家知识和经验，融合行业标准和最佳实践，通过多元数据库统一纳管，实现标准化、自动化、智能化的数据库全生命周期管理。

本文将详细介绍 zCloud 中 TiDB 相关管理组件的使用技巧。

![zCloud.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/z_Cloud_139737eb8e.png)

## 1. TiDB 集群管理

基本信息展示：查看 TiDB 集群服务基本信息，包括集群数量、系列、数据库池、 所属项目组、创建时间、集群列表、以及集群拓扑图等信息

![TiDB集群管理 1.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/Ti_DB_1_3e8d7b96f9.png)

集群详情：展示集群 ID、TiDB 实例 IP、端口、版本、状态、主机数量、PD 数量、TiDB 数量、TiKV 数量、Tiflash 数量及创建时间等信息

![TiDB集群管理 2.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/Ti_DB_2_ed33ebcbaa.png)

## 2.性能监控

性能监控列表可分别查看 TiDB、PD、TiKV 三项标签，展示服务中文名称、服务名称、角色/集群、实例数量、集群状态、监控状态、CPU 使用率展示等各项信息

TiDB 性能列表：

![TiDB 性能列表.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/Ti_DB_c8660c35e3.png)

PD 性能列表：

![PD 性能列表.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/PD_2417e863df.png)

TiKV 性能列表

![TiKV 性能列表.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/Ti_KV_248c47eebc.png)

此外，各个实例性能图表汇总了 TiDB 各项监控指标项，各个性能图表之间可以联动展示各项指标数据（以下仅展示 TiDB 部分性能图表）

![TiDB 性能列表示意.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/Ti_DB_6d1ea23d07.png)

## 3.告警展示

zCloud 平台已预设了多个 TiDB 告警项，并且支持自定义创建、修改各项告警检查信息，如告警项通知设置、告警项阈值设置、告警响应/超时通知设置等

![告警展示.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/_3153a16457.png)

## 4.智能巡检

zCloud 平台已预设了多个的 TiDB 巡检项并支持自定义创建、修改、删除巡检项

![智能巡检.png](https://www-website-strapi.oss-cn-shanghai.aliyuncs.com/prod/_1d63bd09c5.png)

zCloud 在后续版本中，还将持续加强对 TiDB 数据库的支持，包括自动化安装部署，智能诊断分析等更多功能！
