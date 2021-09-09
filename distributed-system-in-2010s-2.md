---
title: 分布式系统 in 2010s ：软件构建方式和演化
author: ['黄东旭']
date: 2019-12-30
summary: 本文为「分布式系统 in 2010s」系列第二篇，内容为软件构建的方式和演化。
tags: ['分布式系统前沿技术']
---

我上大学的时候专业是软件工程，当时的软件工程是 CMM、瀑布模型之类。十几年过去了，看看现在我们的软件开发模式，尤其是在互联网行业，敏捷已经成为主流，很多时候老板说业务下周上线，那基本就是怎么快怎么来，所以现代架构师对于可复用性和弹性会有更多的关注。我所知道业界对 SOA 的关注是从 Amazon 的大规模 SOA 化开始， 2002 年 Bezos 要求 Amazon 的工程团队将所有的业务 API 和服务化，[几条原则](https://www.cio.com/article/3218667/have-you-had-your-bezos-moment-what-you-can-learn-from-amazon.html)放在今天仍然非常适用：

>- All teams will henceforth expose their data and functionality through service interfaces.
>
>- Teams must communicate with each other through these interfaces.
>
>- There will be no other form of inter-process communication allowed: no direct linking, no direct reads of another team’s data store, no shared-memory model, no back-doors whatsoever. The only communication allowed is via service interface calls over the network.
>
>- It doesn’t matter what technology they use.
>
>- All service interfaces, without exception, must be designed from the ground up to be externalizable. That is to say, the team must plan and design to be able to expose the interface to developers in the outside world. No exceptions.

尤其最后一条，我个人认为对于后来的 AWS 的诞生有直接的影响，另外这条也间接地对工程团队的软件质量和 API 质量提出了更高的要求。亚马逊在 SOA 上的实践是组件化在分布式环境中的延伸，尽可能地将业务打散成最细粒度的可复用单元（Services），新的业务通过组合的方式构建。这样的原则一直发展到今天，我们提到的微服务、甚至 Serverless，都是这个思想的延伸。

## SOA 只是一个方法论

很多人在思考 SOA 和微服务的区别时，经常有一些观点类似：「拆的粗就是 SOA，拆的细就是微服务 」，「使用 RESTful API 就是微服务，用 RPC 是 SOA」，「使用 XXX（可以是任何流行的开源框架） 的是微服务，使用 YYY 的是 SOA」... 这些观点我其实并不认可，我理解的 SOA 或者微服务只是一个方法论，核心在于有效地拆分应用，实现敏捷构建和部署，至于使用什么技术或者框架其实无所谓，甚至 SOA 本身就是反对绑定在某项技术上的。

对于架构师来说， 微服务化也并不是灵丹妙药，有一些核心问题，在微服务化的实践中经常会遇到：

1. 服务的拆分粒度到底多细？

2. 大的单体服务如何避免成为单点，如何支持快速的弹性水平扩展？

3. 如何进行流控和降级？防止调用者 DDoS？

4. 海量服务背景下的 CI/CD (测试，版本控制，依赖管理)，运维（包括 tracing，分布式 metric 收集，问题排查）

    … …

上面几个问题都很大。熟悉多线程编程的朋友可能比较熟悉 Actor 模型，我认为 Actor 的思想和微服务还是很接近的，同样的最佳实践也可以在分布式场景下适用，事实上 Erlang OTP 和 Scala 的 Akka Framework 都尝试直接将 Actor 模型在大规模分布式系统中应用。其实在软件工程上这个也不是新的东西，Actor 和 CSP 的概念几乎在软件诞生之初就存在了，现在服务化的兴起我认为是架构复杂到一定程度后很自然的选择，就像当年 CSP 和 Actor 简化并发编程一样。

## 服务化和云

从服务化的大方向和基础设施方面来说，我们这几年经历了：本地单体服务 + 私有 API （自建数据中心，自己运维管理） -> 云 IaaS + 本地服务 + 云提供的 Managed Service (例如 EC2 + RDS)  ->  Serverless 的转变。其本质在于云的出现让开发者对于硬件控制力越来越低，算力和服务越来越变成标准化的东西。而容器的诞生，使得资源复用的粒度进一步的降低（物理机 -> VM -> Container），这无疑是云厂商非常希望看到的。对公有云厂商来说，资源分配的粒度越细越轻量，就越能精准地分配，以提升整体的硬件资源利用率，实现效益最大化。

这里暗含着一个我的观点：公有云和私有云在价值主张和商业模式上是不一样的：对公有云来说，只有不断地规模化，通过不断提升系统资源的利用率，获取收益（比如主流的公有云几乎对小型实例都会超卖）。而私有云的模式可以概括成降低运维成本（标准化服务 + 自动化运维），对于自己拥有数据中心的企业来说，通过云技术提升硬件资源的利用率是好事，只是这个收益并没有公有云的规模化收益来得明显。

在服务化的大背景下，也产生了另外一个趋势，就是基础软件的垂直化和碎片化，当然这也是和现在的 workload 变得越来越大，单一的数据库软件或者开发框架很难满足多变且极端的需求有关。数据库、对象存储、RPC、缓存、监控这几个大类，几乎每位架构师都熟悉多个备选方案，根据不同需求排列组合，一个 Oracle 包打天下的时代已经过去了。

这样带来的结果是数据或状态在不同系统之间的同步和传递成为一个新的普遍需求，这就是为什么以 Kafka，Pulsar 为代表的分布式的消息队列越来越流行。但是在异构数据源之间的同步，暗含了异步和不一致（如果需要一致性，那么就需要对消费者实现幂等的语义），在一些对一致性有极端需求的场景，仍然需要交给数据库处理。

在这种背景下，容器的出现将计算资源分配的粒度进一步的降低且更加标准化，硬件对于开发者来说越来越透明，而且随着 workload 的规模越来越大，就带来的一个新的挑战：海量的计算单元如何管理，以及如何进行服务编排。既然有编排这里面还隐含了另外一个问题：服务的生命周期管理。

## Kubernetes 时代开始了

其实在 Kubernetes 诞生之前，很多产品也做过此类尝试，例如 Mesos。Mesos 早期甚至并不支持容器，主要设计的目标也是短任务（后通过 Marathon Framework 支持长服务），更像一个分布式的工作流和任务管理（或者是分布式进程管理）系统，但是已经体现了 Workload 和硬件资源分离的思想。

在前 Kubernetes 时代，Mesos 的设计更像是传统的系统工程师对分布式任务调度的思考和实践，而 K8s 的野心更大，从设计之初就是要在硬件层之上去抽象所有类型的 workload，构建自己的生态系统。如果说 Mesos 还是个工具的话，那么 K8s 的目标其实是奔着做一个分布式操作系统去的。简单做个类比：整个集群的计算资源统一管控起来就像一个单机的物理计算资源，容器就像一个个进程，Overlay network 就像进程通信，镜像就像一个个可执行文件，Controller 就像 Systemd，Kubectl 就像 Shell……同样相似的类比还有很多。

从另一方面看，Kubernetes 为各种 IaaS 层提供了一套标准的抽象，不管你底层是自己的数据中心的物理机，还是某个公有云的 VM，只要你的服务是构建在 K8s 之上，那么就获得了无缝迁移的能力。K8s 就是一个更加中立的云，在我的设想中，未来不管是公有云还是私有云都会提供标准 K8s 能力。对于业务来说，基础架构的上云，最安全的路径就是上 K8s，目前从几个主流的公有云厂商的动作上来看（GCP 的 GKE，AWS 的 EKS，Azure 的 AKS），这个假设是成立的。

不选择 K8s 的人很多时候会从性能角度来攻击 K8s，理由是：多一层抽象一定会损害性能。对于这个我是不太同意的。从网络方面说，大家可能有个误解，认为 Overlay Network 的性能一定不好，其实这不一定是事实。下面这张图来自 ITNEXT 的工程师对几个流行的 CNI 实现的[评测](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-36475925a560)：

![图 1 Kubernetses CNI benchmark](media/distributed-system-in-2010s-2/1.png)

<div class="caption-center"> Kubernetses CNI benchmark</div>

我们其实可以看到，除了 WaveNet Encrypted 因为需要额外的加密导致性能不佳以外，其它的 CNI 实现几乎已经和 Bare metal 的 host network 性能接近，出现异常的网络延迟大多问题是出现在 iptable NAT 或者 Ingress 的错误配置上面。

所以软件的未来在哪里？我个人的意见是硬件和操作系统对开发者会更加的透明，也就是现在概念刚开始普及起来的 Serverless。我经常用的一个比喻是：如果自己维护数据中心，采购服务器的话，相当于买房；使用云 IaaS 相当于租房；而 Serverless，相当于住酒店。长远来看，这三种方案都有各自适用的范围，并不是谁取代谁的关系。目前看来 Serverless 因为出现时间最短，所以发展的潜力也是最大的。

从服务治理上来说，微服务的碎片化必然导致了管理成本上升，所以近年 Service Mesh （服务网格）的概念才兴起。 服务网格虽然名字很酷，但是其实可以想象成就是一个高级的负载均衡器或服务路由。比较新鲜的是 Sidecar 的模式，将业务逻辑和通信解耦。我其实一直相信未来在七层之上，会有一层以 Service Mesh 和服务为基础的「八层网络」，不过目前并没有一个事实标准出现。Istio 的整体架构过于臃肿，相比之下我更加喜欢单纯使用 Envoy 或者 Kong 这样更加轻量的 API Proxy。 不过我认为目前在 Service Mesh 领域还没有出现有统治地位的解决方案，还需要时间。

>本文是「分布式系统前沿技术」专题文章，目前该专题在持续更新中，欢迎大家保持关注。
