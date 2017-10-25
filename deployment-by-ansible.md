---
title: 使用 Ansible 安装部署 TiDB
author: ['刘博']
date: 2017-06-08
summary: 作为一个分布式系统，在多个节点分别配置安装服务会相当繁琐。Ansible 是基于 Python 的自动化运维工具，糅合了众多老牌运维工具的优点实现了批量操作系统配置、批量程序的部署、批量运行命令等功能，而且使用简单，仅需在管理工作站上安装 Ansible 程序配置被管控主机的 IP 信息，被管控的主机无客户端。选用自动化工具 Ansible 来批量的安装、配置、部署 TiDB 。本文介绍如何通过 Ansible 工具来批量安装，使整个过程简单化。
tags: ['Ansible', 'TiDB']
---

## 背景知识
TiDB 作为一个分布式数据库，在多个节点分别配置安装服务会相当繁琐，为了简化操作以及方便管理，使用自动化工具来批量部署成为了一个很好的选择。

Ansible 是基于 Python 研发的自动化运维工具，糅合了众多老牌运维工具的优点实现了批量操作系统配置、批量程序的部署、批量运行命令等功能，而且使用简单，仅需在管理工作站上安装 Ansible 程序配置被管控主机的 IP 信息，被管控的主机无客户端。基于以上原因，我们选用自动化工具 Ansible 来批量的安装配置以及部署 TiDB。

下面我们来介绍如何使用 Ansible 来部署 TiDB。

## TiDB 安装环境配置如下
操作系统使用 CentOS7.2 或者更高版本，文件系统使用 EXT4。

> 说明：低版本的操作系统(例如 CentOS6.6 )和 XFS 文件系统会有一些内核 Bug，会影响性能，我们不推荐使用。

| IP            | Services                                        |
|:--------------|:------------------------------------------------|
| 192.168.1.101 | PD Prometheus Grafana Pushgateway Node_exporter |
| 192.168.1.102 | PD TiDB Node_exporter                           |
| 192.168.1.103 | PD TiDB Node_exporter                           |
| 192.168.1.104 | TiKV Node_exporter                              |
| 192.168.1.105 | Tikv Node_exporter                              |
| 192.168.1.106 | TiKV Node_exporter                              |

我们选择使用 3 个 PD、2 个 TiDB、3 个 TiKV，这里简单说一下为什么这样部署。

+ 对于 PD 。PD 本身是一个分布式系统，由多个节点构成一个整体，并且同时有且只有一个主节点对外提供服务。各个节点之间通过选举算法来确定主节点，选举算法要求节点个数是奇数个 (2n+1) ，1 个节点的风险比较高，所以我们选择使用 3 个节点。
+ 对于 TiKV 。TiDB 底层使用分布式存储，我们推荐使用奇数 (2n+1) 个备份，挂掉 n 个备份之后数据仍然可用。使用 1 备份或者 2 备份的话，有一个节点挂掉就会造成一部分数据不可用，所以我们选择使用 3 个节点、设置 3 个备份 (默认值)。
+ 对于 TiDB 。我们的 TiDB 是无状态的，现有集群的 TiDB 服务压力大的话，可以在其他节点直接增加 TiDB 服务，无需多余的配置。我们选择使用两个 TiDB，可以做 HA 和负载均衡。
+ 当然如果只是测试集群的话，完全可以使用一个 PD 、一个 TiDB 、三个 TiKV (少于三个的话需要修改备份数量)

## 下载 TiDB 安装包并解压

```
#创建目录用来存放 ansible 安装包
mkdir /root/workspace

#切换目录
cd /root/workspace

#下载安装包
wget https://github.com/pingcap/tidb-ansible/archive/master.zip

#解压压缩包到当前目录下
unzip master.zip

#查看安装包结构，主要内容说明如下
cd tidb-ansible-master && ls
```

**部分内容含义**

```
ansible.cfg: ansible 配置文件
inventory.ini: 组和主机的相关配置
conf: TiDB 相关配置模版
group_vars: 相关变量配置
scripts: grafana 监控 json 模版
local_prepare.yml: 用来下载相关安装包
bootstrap.yml: 初始化集群各个节点
deploy.yml: 在各个节点安装 TiDB 相应服务
roles: ansible tasks 的集合
start.yml: 启动所有服务
stop.yml: 停止所有服务
unsafe_cleanup_data.yml: 清除数据
unsafe_cleanup.yml: 销毁集群
```

## 修改配置文件
主要配置集群节点的分布情况，以及安装路径。

会在 tidb\_servers 组中的机器上安装 TiDB 服务(其他类似)，默认会将所有服务安装到变量 deploy_dir 路径下。

```
#将要安装 TiDB 服务的节点
[tidb_servers]
192.168.1.102
192.168.1.103

#将要安装 TiKV 服务的节点
[tikv_servers]
192.168.1.104
192.168.1.105
192.168.1.106

#将要安装 PD 服务的节点
[pd_servers]
192.168.1.101
192.168.1.102
192.168.1.103

#将要安装 Promethues 服务的节点
# Monitoring Part
[monitoring_servers]
192.168.1.101

#将要安装 Grafana 服务的节点
[grafana_servers]
192.168.1.101

#将要安装 Node_exporter 服务的节点
[monitored_servers:children]
tidb_servers
tikv_servers
pd_servers

[all:vars]
#服务安装路径，每个节点均相同，根据实际情况配置
deploy_dir = /home/tidb/deploy

## Connection
#方式一：使用 root 用户安装
# ssh via root:
# ansible_user = root
# ansible_become = true
# ansible_become_user = tidb

#方式二：使用普通用户安装(需要有 sudo 权限)
# ssh via normal user
ansible_user = tidb

#集群的名称，自定义即可
cluster_name = test-cluster

# misc
enable_elk = False
enable_firewalld = False
enable_ntpd = False

# binlog trigger
#是否开启 pump，pump 生成 TiDB 的 binlog
#如果有从此 TiDB 集群同步数据的需求，可以改为 True 开启
enable_binlog = False
```

安装过程可以分为 root 用户安装和普通用户安装两种方式。有 root 用户当然是最好的，修改系统参数、创建目录等不会涉及到权限不够的问题，能够直接安装完成。
但是有些环境不会直接给 root 权限，这种场景就需要通过普通用户来安装。为了配置简便，我们建议所有节点都使用相同的普通用户；为了满足权限要求，我们还需要给这个普通用户 sudo 权限。
**下面介绍两种安装方式的详细过程，安装完成之后需要手动启动服务。**

#### 1. 使用 root 用户安装
+ 下载 Binary 包到 downloads 目录下，并解压拷贝到 resources/bin 下，之后的安装过程就是使用的 resources/bin 下的二进制程序

```
ansible-playbook -i inventory.ini local_prepare.yml
```

+ 初始化集群各个节点。会检查 inventory.ini 配置文件、Python 版本、网络状态、操作系统版本等，并修改一些内核参数，创建相应的目录。
    - 修改配置文件如下

    ```
    ## Connection
    # ssh via root:
    ansible_user = root
    # ansible_become = true
    ansible_become_user = tidb

    # ssh via normal user
    # ansible_user = tidb
    ```

    - 执行初始化命令

    ```
    ansible-playbook -i inventory.ini bootstrap.yml -k   #ansible-playboo命令说明请见附录
    ```

+ 安装服务。该步骤会在服务器上安装相应的服务，并自动设置好配置文件和所需脚本。
    - 修改配置文件如下

    ```
    ## Connection
    # ssh via root:
      ansible_user = root
      ansible_become = true
      ansible_become_user = tidb

    # ssh via normal user
    # ansible_user = tidb
    ```

    - 执行安装命令

    ```
    ansible-playbook -i inventory.ini deploy.yml -k
    ```

#### 2. 使用普通用户安装
+ 下载 Binary 包到中控机

```
ansible-playbook -i inventory.ini local_prepare.yml
```

+ 初始化集群各个节点。
    - 修改配置文件如下

    ```
    ## Connection
    # ssh via root:
    # ansible_user = root
    # ansible_become = true
    # ansible_become_user = tidb

    # ssh via normal user
    ansible_user = tidb
    ```

    - 执行初始化命令

    ```
    ansible-playbook -i inventory.ini bootstrap.yml -k -K
    ```

+ 安装服务

```
ansible-playbook -i inventory.ini deploy.yml -k -K
```

## 启停服务
+ 启动所有服务

```
ansible-playbook -i inventory.ini start.yml -k
```

+ 停止所有服务

```
ansible-playbook -i inventory.ini stop.yml
```

*附录*

> **ansible-playbook -i inventory.ini xxx.yml -k -K**
>
>   -k 执行之后需要输入 ssh 连接用户的密码，如果做了中控机到所有节点的互信，则不需要此参数
>
>   -K 执行之后需要输入 sudo 所需的密码，如果使用 root 用户或者 sudo 无需密码，则不需要此参数
