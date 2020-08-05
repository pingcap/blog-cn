---
title: TiDB 最佳实践系列（六）HAProxy 的使用
author: ['李仲舒']
date: 2019-11-19
summary: TiDB Server 作为无限水平扩展的无状态计算节点，需要能提供稳定且高性能的负载均衡组件用对外统一的接口地址来提供服务，而 HAProxy 在负载均衡的生态中占有很大的市场。本文将介绍在 TiDB 下使用 HAProxy 的最佳实践。
tags: ['HAProxy','最佳实践','TiDB']
---

HAProxy 是一个使用 C 语言编写的自由及开放源代码软件，其提供高可用性、负载均衡，以及基于 TCP 和 HTTP 的应用程序代理。GitHub、Bitbucket、Stack Overflow、Reddit、Tumblr、Twitter 和 Tuenti 在内的知名网站，及亚马逊网络服务系统都在使用 HAProxy。

TiDB Server 作为无限水平扩展的无状态计算节点，需要能提供稳定且高性能的负载均衡组件用对外统一的接口地址来提供服务，而 HAProxy 在负载均衡的生态中占有很大的市场，TiDB 用户可以将这一成熟稳定的开源工具应用在自己的线上业务中，承担负载均衡、高可用的功能。

![图 1 部署架构](media/best-practice-haproxy/部署架构.jpg)

## HAProxy 简介

HAProxy 由 Linux 内核的核心贡献者 Willy Tarreau 于 2000 年编写，他现在仍然负责该项目的维护，并在开源社区免费提供版本迭代。最新的稳定版本 2.0.0 于 2019 年 8 月 16 日发布，带来更多 [优秀的特性](https://www.haproxy.com/blog/haproxy-2-0-and-beyond/)。

## HAProxy 部分核心功能

*   [高可用性](http://cbonte.github.io/haproxy-dconv/1.9/intro.html#3.3.4)：HAProxy 提供优雅关闭服务和无缝切换的高可用功能；

*   [负载均衡](http://cbonte.github.io/haproxy-dconv/1.9/configuration.html#4.2-balance)：L4（TCP）和 L7（HTTP）负载均衡模式，至少 9 类均衡算法，比如 roundrobin，leastconn，random 等；

*   [健康检查](http://cbonte.github.io/haproxy-dconv/1.9/configuration.html#5.2-check)：对 HAProxy 配置的 HTTP 或者 TCP 模式状态进行检查；

*   [会话保持](http://cbonte.github.io/haproxy-dconv/1.9/intro.html#3.3.6)：在应用程序没有提供会话保持功能的情况下，HAProxy 可以提供该项功能；

*   [SSL](http://cbonte.github.io/haproxy-dconv/1.9/intro.html#3.3.2)：支持 HTTPS 通信和解析；

*   [监控与统计](http://cbonte.github.io/haproxy-dconv/1.9/intro.html#3.3.3)：通过 web 页面可以实时监控服务状态以及具体的流量信息。


## HAProxy 部署操作

### 1. 硬件要求 

根据 [HAProxy 官方文档](http://cbonte.github.io/haproxy-dconv/2.0/management.html#1) 对 HAProxy 的服务器硬件配置有以下建议（也可以根据负载均衡环境进行实际推算，在此基础上提高服务器配置）：

|硬件资源|最低配置|
|:---|:---|
|CPU|2 核，3.5 GHz|
|内存|16 GB|
|存储容量|50 GB（SATA 盘）|
|网卡|万兆网卡|

### 2. 软件要求

根据官方介绍，我们对操作系统和依赖包有以下建议（如果是通过 yum 源部署安装 HAProxy 软件，依赖包可以不需要单独安装）：

#### 操作系统

- Linux 2.4 操作系统，支持 x86、x86_64、Alpha、SPARC、MIPS 和 PA-RISC 架构。
- Linux 2.6 或 3.x 操作系统，支持 x86、x86_64、ARM、SPARC 和 PPC64 架构。
- Solaris 8 或 9 操作系统，支持 UltraSPARC II 和 UltraSPARC III 架构。
- Solaris 10 操作系统，支持 Opteron 和 UltraSPARC 架构。
- FreeBSD 4.10~10 操作系统，支持 x86 架构。
- OpenBSD 3.1 及以上版本操作系统，支持 i386、AMD64、macppc、Alpha 和 SPARC64 架构。
- AIX 5.1~5.3 操作系统，支持 Power™ 架构。

#### 依赖包

- epel-release
- gcc
- systemd-devel


### 3. 推荐版本

根据官方建议，目前 HAProxy 稳定版本为稳定版 2.0，特性介绍参考 [这篇文章](https://www.haproxy.com/blog/haproxy-2-0-and-beyond/)。

### 4.操作步骤

HAProxy 配置 Database 负载均衡场景操作简单，以下 step by step 操作具有普遍性，不具有特殊性，建议根据实际场景，个性化配置相关的配置文件。

1. 安装 HAProxy：推荐 yum 安装
	
	```
	# yum 安装 HAProxy
	yum -y install haproxy
	# 验证 HAProxy 安装是否成功
	which haproxy
	```
	
2.  配置 HAProxy

	```
	# yum 安装过程中会生成配置模版
	vim /etc/haproxy/haproxy.cfg
	```

3.  启动  HAProxy

	方法一：直接启动
	
	```
	haproxy -f /etc/haproxy/haproxy.cfg
	```

	方法二：systemd 启动 HAProxy，默认读取（推荐）
	
	```
	systemctl start haproxy.service
	```

4.  停止  HAProxy

	方法一：kill -9

	```
	ps -ef | grep haproxy 
	kill -9 haproxy.pid
	```
	
	方法二：systemd 停止 HAProxy（如果使用 systemd 启动）

	```
	systemctl stop haproxy.service
	```

## HAProxy 命令介绍

通过以下命令查看 HAProxy 的命令列表：

```
$ haproxy --help
Usage : haproxy [-f <cfgfile|cfgdir>]* [ -vdVD ] [ -n <maxconn> ] [ -N <maxpconn> ]
        [ -p <pidfile> ] [ -m <max megs> ] [ -C <dir> ] [-- <cfgfile>*]
```

|参数|描述|
|:-----|:-----|
|-v|显示简略的版本信息。|
|-vv|显示详细的版本信息。|
|-d|debug 模式开启。|
| -db|仅禁止后台模式|
|-dM [&lt;byte&gt;]|执行分配内存。|
|-V|启动过程显示配置和轮询信息。|
|-D|开启守护进程模式。|
|-C &lt;dir&gt;|在加载配置文件之前更改目录位置。|
|-W|主从模式。|
|-q|静默模式，不输出信息。|
|-c|只检查配置文件并在尝试绑定之前退出。|
|-n|设置最大总连接数为 2000。 |
|-m|限制最大可用内存（单位：MB）。|
|-N|设置单点最大连接数，默认为 2000。 |
|-L|本地实例对等名称。|
|-p|将 HAProxy 所有子进程的 PID 信息写入该文件。 |
|-de|禁止使用 speculative epoll，epoll 仅在 Linux 2.6 和某些定制的 Linux 2.4 系统上可用。|
|-dp|禁止使用 epoll，epoll 仅在 Linux 2.6 和某些定制的 Linux 2.4 系统上可用。|
|-dS|禁止使用 speculative epoll，epoll 仅在 Linux 2.6 和某些定制的 Linux 2.4 系统上可用。|
|-dR|禁止使用 SO_REUSEPORT。|
|-dr|忽略服务器地址解析失败。|
|-dV|禁止在服务器端使用 SSL。|
|-sf/-st &lt;unix_socket&gt; |在启动后，在 pidlist 中发送 FINISH 信号给 PID。收到此信号的进程将等待所有会话在退出之前完成，即优雅停止服务。此选项必须最后指定，后跟任意数量的 PID，SIGTTOU 和 SIGUSR1 都被发送。|
|-x &lt;unix_socket&gt;,[&lt;bind options&gt;...]|获取 socket 信息。|
|-S &lt;unix_socket&gt;,[&lt;bind options&gt;...]|分配新的 socket。|


## HAProxy 最佳实践

```yaml
global                                     # 全局配置
   log         127.0.0.1 local0            # 定义全局的 syslog 服务器，最多可以定义两个
   chroot      /var/lib/haproxy            # 将当前目录为指定目录，设置超级用户权限启动进程，提高安全性
   pidfile     /var/run/haproxy.pid        # 将 HAProxy 进程写入 PID 文件
   maxconn     4000                        # 设置每个 HAProxy 进程锁接受的最大并发连接数
   user        haproxy                     # 同 uid 参数，使用是用户名
   group       haproxy                     # 同 gid 参数，建议专用用户组
   nbproc      40                          # 启动多个进程来转发请求，需要调整到足够大的值来保证 HAProxy 本身不会成为瓶颈
   daemon                                  # 让 HAProxy 以守护进程的方式工作于后台，等同于“-D”选项的功能。当然，也可以在命令行中用“-db”选项将其禁用。
   stats socket /var/lib/haproxy/stats     # 定义统计信息保存位置

defaults                                   # 默认配置
   log global                              # 日志继承全局配置段的设置
   retries 2                               # 向上游服务器尝试连接的最大次数，超过此值就认为后端服务器不可用
   timeout connect  2s                     # HAProxy 与后端服务器连接超时时间，如果在同一个局域网内可设置成较短的时间
   timeout client 30000s                   # 定义客户端与 HAProxy 连接后，数据传输完毕，不再有数据传输，即非活动连接的超时时间
   timeout server 30000s                   # 定义 HAProxy 与上游服务器非活动连接的超时时间

listen admin_stats                         # frontend 和 backend 的组合体，监控组的名称，按需自定义名称
   bind 0.0.0.0:8080                       # 配置监听端口
   mode http                               # 配置监控运行的模式，此处为 `http` 模式
   option httplog                          # 表示开始启用记录 HTTP 请求的日志功能
   maxconn 10                              # 最大并发连接数
   stats refresh 30s                       # 配置每隔 30 秒自动刷新监控页面
   stats uri /haproxy                      # 配置监控页面的 URL
   stats realm HAProxy                     # 配置监控页面的提示信息
   stats auth admin:pingcap123             # 配置监控页面的用户和密码 admin，可以设置多个用户名
   stats hide-version                      # 配置隐藏统计页面上的 HAProxy 版本信息
   stats  admin if TRUE                    # 配置手工启用/禁用，后端服务器（HAProxy-1.4.9 以后版本）

listen tidb-cluster                        # 配置 database 负载均衡
   bind 0.0.0.0:3390                       # 配置浮动 IP 和 监听端口
   mode tcp                                # HAProxy 中要使用第四层的应用层
   balance leastconn                       # 连接数最少的服务器优先接收连接。`leastconn` 建议用于长会话服务，例如 LDAP、SQL、TSE 等，而不是短会话协议，如 HTTP。该算法是动态的，对于实例启动慢的服务器，权重会在运行中作调整。
   server tidb-1 10.9.18.229:4000 check inter 2000 rise 2 fall 3       # 检测 4000 端口，检测频率为 2000 毫秒。如果检测出 2 次正常就认定机器已恢复正常使用，如果检测出 3 次失败便认定该服务器不可用。
   server tidb-2 10.9.39.208:4000 check inter 2000 rise 2 fall 3
   server tidb-3 10.9.64.166:4000 check inter 2000 rise 2 fall 3
```

## 总结

本文介绍了在 TiDB 下使用 HAProxy 的最佳实践，全文对于 HAProxy 的基本使用方法进行较为详细的介绍，这里唯一遗憾的是没有将 HAProxy 的高可用架构和方案加以文字描述，大家在线上使用中可以通过 Linux 的 Keepalived 来实现主备配置，实现 HAProxy 的高可用；在按照该文档搭建 HAProxy 时候，一定要结合自己的具体业务需求和场景，适当调整参数，为业务的负载均衡和可用性提供最佳的保障方案。

最后也希望活跃在 TiDB 社区的小伙伴可以踊跃分享最佳实践经验，大家可以在 TiDB User Group 问答论坛交流讨论使用技巧（[https://asktug.com/](https://asktug.com/)）。
