---
title: DM 源码阅读系列文章（一）序
author: ['杨非']
date: 2019-03-19
summary: 本篇文章主要介绍了 DM 源码阅读的目的和源码阅读的规划，简单介绍了 DM 的源码结构和工具链。本文为本系列文章的第一篇。
tags: ['DM 源码阅读','社区']
---

## 前言

[TiDB Data Migration](https://github.com/pingcap/dm) 是由 PingCAP 开发的一体化数据同步任务管理平台，支持从 MySQL 或 MariaDB 到 TiDB 的全量数据迁移和增量数据同步，在 [TiDB DevCon 2019](https://pingcap.com/community-cn/devcon2019/) 正式开源。作为一款连接 MySQL/MariaDB 生态和 TiDB 生态的中台类型产品，DM 获得了广泛的关注，很多公司、开发者和社区的伙伴已经在使用 DM 来进行数据迁移和管理。随着大家使用的广泛和深入，遇到了不少由于对 DM 原理不理解而错误使用的情况，也发现了一些 DM 支持并不完善的场景和很多可以改进的地方。

在这样的背景下，我们希望开展 DM 源码阅读分享活动，通过对 DM 代码的分析和设计原理的解读，帮助大家理解 DM 的实现原理，和大家进行更深入的交流，也有助于我们和社区共同进行 DM 的设计、开发和测试。

## 背景知识

本系列文章会聚焦 DM 自身，读者需要有一些基本的知识，包括但不限于：

*   Go 语言，DM 由 Go 语言实现，有一定的 Go 语言基础有助于快速理解代码。

*   数据库基础知识，包括 MySQL、TiDB 的功能、配置和使用等；知道基本的 DDL、DML 语句和事务的基本常识；MySQL 数据备份、主从同步的原理等。

*   基本的后端服务知识，比如后台服务进程管理、RPC 工作原理等。

总体而言，读者需要有一定 MySQL/TiDB 的使用经验，了解 MySQL 数据备份和主从同步的原理，以及可以读懂 Go 语言程序。在阅读 DM 源码之前，可以先从阅读[《TiDB Data Migration 架构设计与实现原理》](https://pingcap.com/blog-cn/tidb-ecosystem-tools-3/)入手，并且参考 [使用文档](https://docs.pingcap.com/zh/tidb-data-migration/v1.0) 在本地搭建一个 DM 的测试环境，从基础原理和使用对 DM 有一个初步的认识，然后再进一步分析源码，深入理解代码的设计和实现。

## 内容概要

源码阅读系列将会从两条线进行展开，一条是围绕 DM 的系统架构和重要模块进行分析，另一条线围绕 DM 内部的同步机制展开分析。源码阅读不仅是对代码实现的分析，更重要的是深入的分析背后的设计思想，源码阅读和原理分析的覆盖范围包括但不限于以下列出的内容（因为目前 DM 仍处于快速迭代的阶段，会有新的功能和模块产生，部分模块在未来也会进行优化和重构，后续源码阅读的内容会随着 DM 的功能演进做适当的调整）：

*   整体架构介绍，包括 DM 有哪些模块，分别实现什么功能，模块之间交互的数据模型和 RPC 实现。

*   DM-worker 内部组件设计原理（relay-unit, dump-unit, load-unit, sync-unit）和数据同步的并发模型设计与实现。

*   基于 binlog 的数据同步模型设计和实现。

*   relay log 的原理和实现。

*   定制化数据同步功能的实现原理（包括库表路由，库表黑白名单，binlog event 过滤，列值转换）。

*   DM 如何支持上游 online DDL 工具（[pt-osc](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html), [gh-ost](https://github.com/github/gh-ost)）的 DDL 同步场景。

*   sharding DDL 处理的具体实现。

*   checkpoint 的设计原理和实现，深入介绍 DM 如何在各类异常情况下保证上下游数据同步的一致性。

*   DM 测试的架构和实现。

## 代码简介

DM 源代码完全托管在 GitHub 上，从 [项目主页](https://github.com/pingcap/dm) 可以看到所有信息，整个项目使用 Go 语言开发，按照功能划分了很多 package，下表列出了 DM 每个 package 的基本功能：

| Package | Introduction |
| :---------- | :----------------------------------------- |
| checker | 同步任务上下游数据库配置、权限前置检查模块 |
| cmd/dm-ctl, cmd/dm-master, cmd/dm-worker | dmctl, DM-master, DM-worker 的 main 文件所在模块 |
| dm/config | 同步任务配置、子任务配置、前置检查配置定义模块 |
| dm/ctl | dmctl 所有 RPC 调用实现的模块 |
| dm/master | DM-master 的核心实现，包含了 DM-master 后台服务，对 dmctl 到 DM-master 的 RPC 调用的处理逻辑，对 DM-worker 的管理，对 sharding DDL 进行协调调度等功能 |
| dm/pb, dm/proto | dm/proto 定义了 DM-master 和 DM-worker 相关交互的 protobuf 协议，dm/pb 是对应的生成代码 |
| dm/unit | 定义了子任务执行的逻辑单元（包括 dump unit, load unit, sync unit, relay unit）接口，在每个不同逻辑单元对应的 package 内都有对应的 接口实现 |
| dm/worker | DM-worker 的核心实现，实现 DM-worker 后台服务，管理维护每个任务的 relay 逻辑单元，管理、调度每个子任务的逻辑单元 |
| loader | 子任务 load 逻辑单元的实现，用于全量数据的导入 |
| mydumper | 子任务 dump 逻辑单元的实现，用于全量数据的导出 |
| pkg | 包含了一些基础功能的实现，例如 gtid 操作、SQL parser 封装、binlog 文件流读写封装等 |
| relay | 处理 relay log 同步的核心模块 |
| syncer | 子任务 sync 逻辑单元的实现，用于增量数据的同步 |

对于理解代码最直接的手段就是从 DM-server, DM-worker 和 dmctl 三个 binary 对应的 main 文件入手，看 DM-worker, DM-master 是如何启动，DM-worker 如何管理一个上游实例和同步任务；如何从 dmctl 开始同步子任务；然后看一个同步子任务从全量状态，到增量同步状态，binlog 如何处理、sql 任务如何分发等。通过这样一个流程对 DM 的整体架构就会有全面的理解。进一步就可以针对每个使用细节去了解 DM 背后的设计逻辑和代码实现，可以从具体每个 package 入手，也可以从感兴趣的功能入手。

实际上 DM 代码中使用了很多优秀的第三方开源代码，包括但不仅限于：

*   借助 [grpc](https://github.com/grpc/grpc-go) 实现各组件之间的 RPC 通信

*   借助 [pingcap/parser](https://github.com/pingcap/parser) 进行 DDL 的语法解析和语句还原

*   借助 [pingcap/tidb-tools](https://github.com/pingcap/tidb-tools) 提供的工具实现复杂的数据同步定制

*   借助 [go-mysql](https://github.com/siddontang/go-mysql) 解析 MySQL/MariaDB binlog 等

在源码阅读过程中对于比较重要的、与实现原理有很高相关度的第三方模块，我们会进行相应的扩展阅读。

## 工具链

工欲善其事，必先利其器，在阅读 DM 源码之前，我们先来介绍 DM 项目使用到的一些外部工具，这些工具通常用于 DM 的构建、部署、运行和测试，在逐步使用 DM，阅读代码、理解原理的过程中都会使用到这些工具。

*   golang 工具链：构建 DM 需要 go >= 1.11.4，目前支持 Linux 和 MacOS 环境。

*   [gogoprotobuf](https://github.com/gogo/protobuf/)：用于从 proto 描述文件生成 protobuf 代码，DM 代码仓库的 [generate-dm.sh](https://github.com/pingcap/dm/blob/master/generate-dm.sh) 文件封装了自动生成 DM 内部 protobuf 代码的脚本。

*   [Ansible](https://docs.ansible.com/)：DM 封装了 [DM-Ansible](https://docs.pingcap.com/zh/tidb-data-migration/v1.0/deploy-a-dm-cluster-using-ansible) 脚本用于 DM 集群的自动化部署，部署流程可以参考 [使用 ansible 部署 DM](https://pingcap.com/docs/tools/dm/deployment/)。

*   [pt-osc](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html), [gh-ost](https://github.com/github/gh-ost)：用于上游 MySQL 进行 online-ddl 的同步场景。

*   [mydumper](https://github.com/pingcap/mydumper)：DM 的全量数据 dump 阶段直接使用 mydumper 的 binary。

*   MySQL, TiDB, sync_diff_inspector：这些主要用于单元测试和集成测试，可以参考 [tests#preparations](https://github.com/pingcap/dm/tree/master/tests#preparations) 这部分描述。

## 小结

本篇文章主要介绍了 DM 源码阅读的目的和源码阅读的规划，简单介绍了 DM 的源码结构和工具链。下一篇文章我们会从 DM 的整体架构入手，详细分析 DM-master、DM-worker 和 dmctl 三个组件服务逻辑的实现和功能抽象，RPC 数据模型和交互接口。更多的代码阅读内容会在后面的章节中逐步展开，敬请期待。
