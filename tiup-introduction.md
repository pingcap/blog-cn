---
title: 从马车到电动车，TiDB 部署工具变形记
author: ['Heng Long']
date: 2020-06-12
summary: 在部署易用性方面，TiDB 开发者经过诸多探索和尝试，经过了命令行时代、Ansible 时代，终于在 TiDB 4.0 发布了新一代具有里程碑意义的解决方案——TiUP。
tags: ['TiUP','安装部署','TiDB 4.0 新特性']
---

打造优秀产品的信念渗透在每一个 TiDB 开发者的血液中，衡量产品的优秀有多个维度：易用性、稳定性、性能、安全性、开放性、拓展性等等。**在部署易用性方面，TiDB 开发者们经过诸多探索和尝试，经过了命令行时代、Ansible 时代，终于在 TiDB 4.0 发布了新一代具有里程碑意义的解决方案——TiUP。**

TiUP 的意义不仅仅在于提供了里程碑式的解决方案，更是对 TiDB 开源社区活力的有力证明。TiUP 从 3 月立项进入 [PingCAP Incubator](https://github.com/pingcap-incubator) 进行孵化，从零开发到最终发布 TiUP 1.0 GA 仅仅只花了两个月。两个月内 40+ 位 Contributor 新增了 690+ 次提交，最终沉淀接近 40k 行代码。

本文会描述整个演进过程，并介绍 TiUP 设计过程中的一些理念和实现细节。

## 以史为鉴

TiUP 的诞生并非一蹴而就，而是有一个演变过程。简要描述这个演变过程，有助于大家更加深入理解 TiUP 的设计和取舍。

### 纯命令行

在没有 TiDB Ansible 的时代，要运行一个 TiDB 集群只能通过命令行的方式。TiDB 集群包含 TiDB/TiKV/PD 三个核心组件， 和 Promethues/Grafana/Node Exporter 监控组件。手动构建一个集群运行需要的所有命令行参数和配置文件比较复杂的。比如，我们想要搭建一个集群，其中启动三个 PD 的命令行参数就有下面这么复杂（可以忽略命令行，仅演示复杂性）：

```
$ bin/pd-server --name=pd-0
--data-dir=data/Rt1J27k/pd-0/data
--peer-urls=http://127.0.0.1:2380
--advertise-peer-urls=http://127.0.0.1:2380 --client-urls=http://127.0.0.1:2379 --advertise-client-urls=http://127.0.0.1:2379 --log-file=data/Rt1J27k/pd-0/pd.log --initial-cluster=pd-0=http://127.0.0.1:2380,pd-1=http://127.0.0.1:2381,pd-2=http://127.0.0.1:2383

$ bin/pd-server --name=pd-1
--data-dir=data/Rt1J27k/pd-1/data
--peer-urls=http://127.0.0.1:2381 --advertise-peer-urls=http://127.0.0.1:2381 --client-urls=http://127.0.0.1:2382 --advertise-client-urls=http://127.0.0.1:2382 --log-file=data/Rt1J27k/pd-1/pd.log --initial-cluster=pd-0=http://127.0.0.1:2380,pd-1=http://127.0.0.1:2381,pd-2=http://127.0.0.1:2383

$ bin/pd-server --name=pd-2
--data-dir=data/Rt1J27k/pd-2/data
--peer-urls=http://127.0.0.1:2383 --advertise-peer-urls=http://127.0.0.1:2383 --client-urls=http://127.0.0.1:2384 --advertise-client-urls=http://127.0.0.1:2384 --log-file=data/Rt1J27k/pd-2/pd.log --initial-cluster=pd-0=http://127.0.0.1:2380,pd-1=http://127.0.0.1:2381,pd-2=http://127.0.0.1:2383
```
>注：以 $ 开头的表示在命令行执行的命令

以上仅仅是启动 PD 就可以发现这种方式显然太复杂、使用门槛太高。尽管我们可以通过把这些东西脚本化，在脚本构建好这些内容，每次执行对应脚本来简化这个过程，但是对于第一次构建脚本的用户来说，也是不小的挑战。

另外在生产环境部署时还需要在多个主机上分发下载对应组件，以及初始化环境，对于扩容又需要各种初始化，相当繁琐。

### TiDB Ansible

第二代方案 [TiDB Ansible](https://github.com/pingcap/tidb-ansible) 基于 [Ansible](https://www.ansible.com/) playbook 功能编写的集群部署工具，简化之后，只需要用户提供拓扑文件，即可提供集群部署和运维功能（启动、关闭、升级、重启、扩容、缩容）。但是 TiDB Ansible 的使用依然非常繁琐，提供的错误消息也不友好，同时只能串行处理，对于大集群的运维和管理尤其不方便。

```
$ vim hosts.ini                                                
$ ansible-playbook -i hosts.ini create_users.yml -u root -k   
$ vim inventory.ini                                        
$ ansible-playbook local_prepare.yml 
$ ansible-playbook bootstrap.yml
$ ansible-playbook deploy.yml
$ ansible-playbook start.yml
```

以上是部署启动一个集群，扩缩容操作更加繁琐。并且由于 Ansible 自身命令执行的特点，整个部署过程的时间较长。

### TiUP

TiUP 在 TiDB Ansible 的基础上进一步对整个集群的部署和运维操作进行了简化。由于 TiUP 从零开发，可以掌控所有实现细节，针对部署 TiDB 集群的需要定制、避免非必需的动作，内部做到最大程度的并行化，同时提供更加友好错误提示。

利用 TiUP 部署集群通过简单的命令即可完成，且执行速度较 TiDB Ansible 大幅提高：

```
$ tiup cluster deploy <cluster-name> <version> <topology.yaml> [flags]   # 部署集群
$ tiup cluster start prod-cluster                                        # 启动集群
```

其他常用的运维操作也同样可以通过一个命令完成：

```
$ tiup cluster scale-in prod-cluster -N 172.16.5.140:20160               # 缩容节点
$ tiup cluster scale-out tidb-test scale.yaml                            # 扩容节点
$ tiup cluster upgrade tidb-test v4.0.0-rc                               # 升级集群
```

## 深入 TiUP

首先使用下面这行脚本安装 TiUP：

```
$ curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

**你会发现，使用 tiup help 命令时，我们并没有 tiup cluster 这个子命令，这是怎么回事儿呢？这就要从 TiUP 的设计理念聊起。**

### TiUP 理念

以上虽然演示了通过 TiUP 快速部署运维集群，但是 TiUP 的定位从来就不是一个运维工具，而是 TiDB 组件管理器。TiUP 之于 TiDB，类似 yum 之于 CentOS，Homebrew 之于 MacOS。

**TiUP 的理念为：简单易用、可扩展、开放、安全。**

### 简单易用

TiUP 本身只包含很少几个命令，几乎不用专门学习、记忆，完全依靠经验和肌肉记忆就能正确使用：

|命令 | 解释 |
|:---------|:--------|
| `tiup install <component>` | 安装一个组件 |
|`tiup uninstall <component>`  | 卸载一个组件|
|  `tiup update` | 更新组件 |
| `tiup list` | 显示组件列表 |
| `tiup status` | 运行的组件状态 | 
| `tiup clean` | 清理组件运行数据 |

### 可扩展

TiUP 最核心之处就是高度可扩展、可定制，除了自带的几个命令之外，还可以通过安装不同的组件，对 TiUP 进行定制，一千个人就有一千种 TiUP，将 TiUP 打造成一把属于自己的瑞士军刀。以上演示 [集群部署运维](https://docs.google.com/document/d/1cuBxlAQ7YGdI-Sy6FYJYY5EjluaBOlXnQZRcesPwrYQ/edit#heading=h.p22ggyeh7snl) 就是通过 TiUP-Cluster 组件提供的功能完成的，除了 TiUP-Cluster 组件之后，还有非常多有用的组件，例如以下两个开发中最常用的组件：

1\. TiUP-Playground 组件，可以一条命令直接运行一个本地 TiDB 集群：

```
$ tiup playground                  # 运行最新稳定版 TiDB 集群
$ tiup playground v3.0.15          # 运行版本为 v3.0.15 TiDB 集群
$ tiup playground --kv 3           # 启动三个 TiKV 节点
$ tiup playground --monitor        # 启动 Prometheus 监控
$ ...

```

2\. TiUP-Bench 组件，可以快速进行基准测试：

```
$ tiup bench tpcc                         # 进行 TPC-C 性能基准测试
$ tiup bench tpch prepare
$ tiup bench tpch run                     # 进行 TPC-H 性能基准测试
```

### 开放

TiUP 的组件模式不仅仅为可扩展性设计的，也希望构建一个开放生态，用户可以根据自己的使用场景，编写自己的组件，并且将组件提交到 TiUP 的镜像仓库。也可以在镜像仓库中根据自己的需求选择由社区开发的组件。

目前 TiUP 镜像仓库已经包含 20+ 组件，希望通过开放的生态基因和 TiDB 庞大的开发者生态创意碰撞，能为 TiUP 提交越来越多优质的组件。

为此，TiUP 提供了一个命令 `tiup mirror publish` 能够将本地组件通过一个命令发布到 TiUP 的镜像仓库。

```
$ tiup mirror publish -h
Publish a component to the repository

Usage:
  tiup mirror publish <comp-name> <version> <tarball> <entry> [flags]

Flags:
      --arch string       the target system architecture (default "amd64")
      --desc string       description of the component
      --endpoint string   endpoint of the server (default "https://tiup-mirrors.pingcap.com/")
  -h, --help              help for publish
  -k, --key string        private key path
      --os string         the target operation system (default "linux")

Global Flags:
      --repo string          Path to the repository
      --skip-version-check   Skip the strict version check, by default a version must be a valid SemVer string
```

### 安全

安全是软件分发系统（组件管理器）的基石，如果分发的组件没有安全保障，那么上面提到简单易用、可扩展、开放都会为恶意软件提供便利。TiUP 作为 TiDB 生态的入口，比如提供企业级安全保障，需要防范在软件分发各个环节中可能出现的各种攻击。

要防范组件分发过程中的各种攻击，需要非常谨慎和精细的设计，得益于 [TUF](https://theupdateframework.com/) 规范的优良设计，我们在 TUF 规范的基础上设计了 TiUP 的软件分发方案，详细的设计文档超过 10 页，本文不会详细讨论所有细节，有兴趣的朋友可以参考 [设计文档](https://github.com/pingcap/tiup/blob/master/doc/design/manifest.md)。

以下是对软件分发过程一个简要的描述（如何在各个环节防范不同类型的攻击，可以参考 TUF 规范和 TiUP 设计文档）：

1.  元信息分级：

	a. root 保存对元信息签名的公钥信息；
	
	b. index 保存各个组件信息和组件 Owner 的公钥信息；
	
	c. component 保存组件的版本信息；

	d. snapshot 保存其他元信息的最新版本号和 Hash 值；

	e. timestamp 保存最新 snapshot 的版本号和 Hash 值。

2.  所有的元信息和组件包在 CDN 是不可变的，不同版本的元信息使用 `${version}.${name}.json` 的文件名保存。

3.  所有组件包的的 Hash 值保存在组件的元信息文件中 `（sha256/sha512）`。

4.  所有的元信息文件都包含该被签名内容和签名信息。

5.  根证书使用 5 个密钥签名，5 个密钥分别由 5 位不同的 TiDB 开发者离线保存。

6.  初始分发的 TiUP 中包含一份由 5 位 TiDB 开发者签名的 `root.json`，后续信息校验会保证 root.json 中至少有三个签名是正确的。

7.  `index/snashot/timestamp` 的不可篡改性由 `root.json` 中的对应的密钥信息保证。

8.  component 的不可篡改性由 `index.json` 中对应的 Owner 密钥保证（社区通过 tiup mirror publish 发布的组件，只有作者拥有私钥）。

9.  各个组件包的不可篡改性由元信息中的 sha256/sha512 Hash 值保证（目前的算力情况下是安全的）。

通过上面的机制，我们能保证用户下载的组件不会经过任何中间环节篡改，从而提供安全的组件分发机制。

希望上面的介绍能让大家对 TiUP 的演进和理念有初步的认识，同时 [TiUP](https://github.com/pingcap/tiup) 开源在 Github 并且随着 TiDB 4.0 GA 版本一起发布，对于 TiUP 有兴趣的小伙伴可以阅读源码，有任何问题都可以通过 Issue 提问或直接在 Slack 的 [#sig-tiup](https://join.slack.com/share/zt-exzk2vc5-72UhEIRqu7Uj_5N5JYb8TQ) 中提问。

### 附：TiUP 贡献者名单

| 序号 | GitHub ID |
| :----|:-----|
| 1 | 5kbpers|
|2| AstroProfundis|
|3|baurine|
|4|birdstorm|
|5|bobotu|
|6|breeswish|
|7|BusyJay|
|8|c4pt0r|
|9|chenlx0|
|10|fewdan|
|11|flowbehappy|
|12|fredchenbj|
|13|hhkbp2|
|14|HunDunDM|
|15|ilovesoup|
|16|JaySon-Huang|
|17|july2993|
|18|King-Dylan|
|19|kissmydb|
|20|kolbe|
|21|lhy1024|
|22|lichunzhu|
|23|liubo0127|
|24|lonng|
|25|lucklove|
|26|mahjonp|
|27|mapleFU|
|28|marsishandsome|
|29|nrc|
|30|overvenus|
|31|qinzuoyan|
|32|rleungx|
|33|siddontang|
|34|sticnarf|
|35|Win-Man|
|36|wjhuang2016|
|37|YangKeao|
|38|yeya24|
|39|yikeke|
|40|zhangjinpeng1987|
|41|zyguan|
