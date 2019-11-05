---
title: 浏览器上可以运行数据库吗？TiDB + WebAssembly 告诉你答案
author: ['刘奇']
date: 2019-11-05
summary: 今天的 TiDB 可以直接运行在浏览器本地。打开浏览器，你可以直接创建数据库，对数据进行增删改查。关掉浏览器，一切都消失了，干净绿色环保
tags: ['wasm', 'SQL', 'Go']
---

一直以来我都有个梦想：

希望有一个数据库能够弹性扩展（分布式）到成百上千节点的规模，易于学习和理解，可以运行在私有云、公有云、Multi-Cloud、Kubernetes，也能够跑在嵌入式设备（比如树莓派）上，更酷的是也能够直接运行在浏览器里，而且不需要任何浏览器扩展（Extension），变成口袋数据库，就像那部电影《蚁人》。

今天，这一切都变成了现实：

**今天的 [TiDB](https://github.com/pingcap/tidb) 可以直接运行在浏览器本地。打开浏览器，你可以直接创建数据库，对数据进行增删改查。关掉浏览器，一切都消失了，干净绿色环保——**

首先在笔记本浏览器打开 [play.pingcap.com](https://play.pingcap.com)（这里用的是 MacOS 上面的 Chrome，不确定其它浏览器是否正常），可能需要几秒来加载页面，然后就能看到熟悉的 Shell 了。现在来试试几个 SQL 语句吧！由于 TiDB 基本兼容 MySQL 协议和语法，因此我们可以用熟悉的 MySQL 风格操作，如下图所示：

![演示-1](media/tidb-in-the-browser-running-a-golang-database-on-wasm/演示-1.gif)

<center>在浏览器上运行 TiDB</center>

**是不是很酷？无痛体验 SQL 的时代到了。**

更酷的是，这一切都运行在浏览器本地，删库再也不用跑路了。

有了这些，那么是时候给在线学习 SQL 教程的网站加点功能了，比如在文字教程时，同步运行 SQL 语句。这里有个简单的 [演示](https://tour.pingcap.com/)：

![SQL 教程网站演示](media/tidb-in-the-browser-running-a-golang-database-on-wasm/演示-2.png)

<center>SQL 教程网站演示</center>

**那么在浏览器里面运行数据库还有哪些好处呢？**

还记得你安装配置数据库的痛苦吗？从此以后，每个人随时随地都可以拥有一个数据库，再没有痛苦的安装过程，再也不用痛苦的配置参数，随时享受写 SQL 的快感。也许我们不再需要 indexdb 了，SQL 是更高级的 API，TiDB 使得「一次编写、到处运行」变成了现实。

当然，你一定很好奇这一切是怎么实现的：

+ 首先要感谢 Go team 让 Go 语言支持了 WebAssembly（Wasm），这是近期最让我兴奋的特性之一，它让在浏览器里运行 Go 语言编写的应用程序成为了现实；

+ 然后感谢 PingCAP 的开源分布式数据库 TiDB。我们把 TiDB 编译成 Wasm，在浏览器里直接运行生成的 Wasm 文件，这就使得在浏览器里运行一个数据库成为了现实。如果没有记错，TiDB 好像是 Go 语言编写的第一个可以在浏览器里面运行的 SQL 数据库；

+ 特别感谢参加 [TiDB Hackathon 2019](https://github.com/pingcap/presentations/blob/master/hackathon-2019/hackathon-2019-projects.md) 的选手和大家各种有趣的想法，尤其感谢 Ti-cool 团队，在他们的努力下这一切变成了现实（该项目也获得了 Hackathon 二等奖）。

**接下来我们可以试试更多有趣的想法：**

+ 让更多的在线 SQL 教程都可以直接运行。
+ 让 TiDB 运行在 Go Playground 上，或许需要 Go team 的帮助。
+ 支持持久化数据库，我们已经有了云计算、边缘计算，为什么不能有浏览器计算呢？
+ ……

还有好多想法我们将在接下来的文章里介绍。如果你有新的、有趣的想法，欢迎 [联系我们](mailto:info@pingcap.com)。

**下一篇文章将由 Ti-cool 团队成员介绍整个项目的实现原理和后续改进工作，敬请期待！如果你已经等不及了，可以在这里直接看 [源码实现](https://github.com/pingcap/tidb/pull/13069)，祝大家玩得开心！**
