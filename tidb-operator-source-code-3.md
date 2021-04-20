---
title: TiDB Operator 源码阅读 (三) 编排组件控制循环
author: ['陈逸文']
date: 2021-04-20
summary: 本篇文章将介绍组件控制循环的编排设计。我们将会了解到完成 TiDB 集群的生命周期管理过程中，各种控制循环事件经过了怎样的编排，这些事件中又完成了哪些资源管理操作。
tags: ['TiDB Operator']
---

[上篇文章](https://pingcap.com/blog-cn/tidb-operator-source-code-2/)中，我们介绍了 TiDB Operator 的 Controller Manager 的设计和实现，了解了各个 Controller 如何接受和处理变更。在这篇文章中，我们将讨论组件的 Controller 的实现。TiDBCluster Controller 负责了 TiDB 主要组件的生命周期管理，我们将以此为例， 介绍组件控制循环的编排设计。我们将会了解到完成 TiDB 集群的生命周期管理过程中，各种控制循环事件经过了怎样的编排，这些事件中又完成了哪些资源管理操作。在阅读时，大家了解这些工作的大致过程和定义即可，我们将在下一篇文章中具体介绍各个组件如何套用下面的范式。

## 组件控制循环的调用

在上一篇文章的代码介绍中，我们提到了 `TiDBCluster controller` 的 `updateTidbCluster` 函数，位于 `pkg/controller/tidbcluster/tidb_cluster_control.go`，它是 TiDB 组件生命周期管理的入口，调用了一系列生命周期管理函数。略去注释，我们可以发现 `updateTidbCluster` 函数依次调用了以下函数：

1. c.reclaimPolicyManager.Sync(tc)

2. c.orphanPodsCleaner.Clean(tc)

3. c.discoveryManager.Reconcile(tc)

4. c.ticdcMemberManager.Sync(tc)

5. c.tiflashMemberManager.Sync(tc)

6. c.pdMemberManager.Sync(tc)

7. c.tikvMemberManager.Sync(tc)

8. c.pumpMemberManager.Sync(tc)

9. c.tidbMemberManager.Sync(tc)

10. c.metaManager.Sync(tc)

11. c.pvcCleaner.Clean(tc)

12. c.pvcResizer.Resize(tc)

13. c.tidbClusterStatusManager.Sync(tc)

这些函数可以分为两类，一是 TiDB 组件的视角组织的控制循环实现，例如 PD，TiDB，TiKV，TiFlash，TiCDC，Pump，Discovery，另外一类是负责管理 TiDB 组件所使用的 Kubernetes 资源的管理以及其他组件外围的生命周期管理操作，例如 PV 的 ReclaimPolicy 的维护，OrphanPod 的清理，Kubernetes 资源的 Meta 信息维护，PVC 的清理和扩容，TiDBCluster 对象的状态管理等。

## TiDB 组件的生命周期管理过程

TiDB 的主要组件控制循环的代码分布在 `pkg/manager/member` 目录下以 `_member_manager.go` 结尾的文件下，比如 `pd_member_manager.go`，这些文件又引用了诸如 `_scaler.go`，`_upgrader.go` 的文件，这些文件包含了扩缩容和升级相关功能的实现。从各个组件的 `_member_manager.go` 相关文件，我们可以提炼出以下通用实现：

```go
// Sync Service
if err := m.syncServiceForTidbCluster(tc); err != nil {
    return err
}
 
// Sync Headless Service
if err := m.syncHeadlessServiceForTidbCluster(tc); err != nil {
    return err
}
 
// Sync StatefulSet
return syncStatefulSetForTidbCluster(tc)
 
func syncStatefulSetForTidbCluster(tc *v1alpha1.TidbCluster) error {
    if err := m.syncTidbClusterStatus(tc, oldSet); err != nil {
        klog.Errorf("failed to sync TidbCluster: [%s/%s]'s status, error: %v", ns, tcName, err)
    }
 
    if tc.Spec.Paused {
        klog.V(4).Infof("tidb cluster %s/%s is paused, skip syncing for statefulset", tc.GetNamespace(), tc.GetName())
        return nil
    }
 
    cm, err := m.syncConfigMap(tc, oldSet)
 
    newSet, err := getnewSetForTidbCluster(tc, cm)
 
    if err := m.scaler.Scale(tc, oldSet, newSet); err != nil {
        return err
    }
 
    if err := m.failover.Failover(tc); err != nil {
        return err
    }
 
    if err := m.upgrader.Upgrade(tc, oldSet, newSet); err != nil {
        return err
    }
 
    return UpdateStatefulSet(m.deps.StatefulSetControl, tc, newSet, oldSet)
}
```

这段代码主要完成了同步 Service 和 同步 Statefulset 的工作，同步 Service 主要是为组件创建或同步 Service 资源，同步 Statefulset 具体包含了一下工作：

1. 同步组件的 Status;

2. 检查 TiDBCluster 是否停止暂停了同步;

3. 同步 ConfigMap;

4. 根据 TidbCluster 配置生成新的 Statefulset，并且对新 Statefulset 进行滚动更新，扩缩容，Failover 相关逻辑的处理;

5. 创建或者更新 Statefulset;

组件的控制循环是对上面几项工作循环执行，使得组件保持最新状态。下面将介绍 TiDB Operator 中这几项工作具体需要完成哪些方面的工作。

## 同步 Service

一般 Service 的 Reconcile 在组件 Reconcile 开始部分，它负责创建和同步组件所使用的 Service，例如 `cluster1-pd` 和 `cluster1-pd-peer`。在控制循环函数中，会调用 `getNewServiceForTidbCluster` 函数，通过 Tidbcluster CR 中记录的信息创建一个新的 Service 的模板，如果 Service 不存在，直接创建 Service，否则，通过比对新老 Service Spec 是否一致，来决定是否要更新 Service 对象。

TiDB 组件使用的 Service 中，包括了 Service 和 Headless Serivce，为组件提供了被访问的能力。当组件不需要知道是哪个实例正在与它通信，并且可以接受负载均衡方式的访问，则可以使用 Service 服务，例如 TiKV，TiDB 等组件访问 PD 时，就可以使用 Service 地址。当组件需要区分是那个 Pod 在提供服务时，则需要用 Pod DNS 进行通信，例如 TiKV 在启动时，会将自己的 [Pod DNS](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-network-id) 作为 Advertise Address 对外暴露，其他 Pod 可以通过这个 Pod DNS 访问到自己。

## 同步 Status

完成 Service 的同步后，组件接入了集群的网络，可以在集群内访问和被访问。控制循环会进入 `syncStatefulSetForTidbCluster`，开始对 Statefulset 进行 Reconcile，首先是使用 `syncTidbClusterStatus` 对组件的 Status 信息进行同步，后续的扩缩容、Failover、升级等操作会依赖 Status 中的信息进行决策。

同步 Status 是 TiDB Operator 比较关键的操作，它需要同步 Kubernetes 各个组件的信息和 TiDB 的集群内部信息，例如在 Kubernetes 方面，在这一操作中会同步集群的副本数量，更新状态，镜像版本等信息，检查 Statefulset 是否处于更新状态。在 TiDB 集群信息方面，TiDB Operator 还需要将 TiDB 集群内部的信息从 PD 中同步下来。例如 PD 的成员信息，TiKV 的存储信息，TiDB 的成员信息等，TiDB 集群的健康检查的操作便是在更新 Status 这一操作内完成。

## 检查 TiDBCluster 是否暂停同步

更新完状态后，会通过 `tc.Spec.Paused` 判断集群是否处于暂停同步状态，如果暂停同步，则会跳过下面更新 Statefulset 的操作。

## 同步 ConfigMap

在同步完 Status 之后，syncConfigMap 函数会更新 ConfigMap，ConfigMap 一般包括了组件的配置文件和启动脚本。配置文件是通过 YAML 中 Spec 的 Config 项提取而来，目前支持 TOML 透传和 YAML 转换，并且推荐 TOML 格式。启动脚本则包含了获取组件所需的启动参数，然后用获取好的启动参数启动组件进程。在 TiDB Operator 中，当组件启动时需要向 TiDB Operator 获取启动参数时，TiDB Operator 侧的信息处理会放到 discovery 组件完成。如 PD 获取参数用于决定初始化还是加入某个节点，就会使用 wget 访问 discovery 拿到自己需要的参数。这种在启动脚本中获取参数的方法，避免了更新 Statefulset 过程中引起的非预期滚动更新，对线上业务造成影响。

## 生成新 Statefulset

`getNewPDSetForTidbCluster` 函数会得到一个新的 Statefulset 的模板，它包含了对刚才生成的 Service，ConfigMap 的引用，并且根据最新的 Status 和 Spec 生成其他项，例如 env，container，volume 等，这个新的 Statefulset 还需要送到下面 3 个流程进行滚动更新，扩缩容，Failover 方面的加工，最后将这个新生成的 Statefulset 会被送到 UpdateStatefulSet 函数处理，判断其是否需要实际更新到组件对应的 Statefulset。

### 新 Statefulset 的加工(一): 滚动更新

`m.upgrader.Upgrade` 函数负责滚动更新相关操作，主要更新 Statefulset 中 `UpgradeStrategy.Type` 和 `UpgradeStrategy.Partition`，滚动更新是借助 Statefulset 的 RollingUpdate 策略实现的。组件 Reconcile 会设置 Statefulset 的升级策略为滚动更新，即组件 Statefulset 的 `UpgradeStrategy.Type` 设置为 RollingUpdate 。在 Kubernetes 的 Statefulset 使用中，可以通过配置 `UpgradeStrategy.Partition` 控制滚动更新的进度，即 Statefulset 只会更新序号大于或等于 partition 的值，并且未被更新的 Pod。通过这一机制就可以实现每个 Pod 在正常对外服务后才继续滚动更新。在非升级状态或者升级的启动阶段，组件的 Reconcile 会将 Statefulset 的 UpgradeStrategy.Partition 设置为 Statefulset 中最大的 Pod 序号，防止有 Pod 更新。在开始更新后，当一个 Pod 更新，并且重启后对外提供服务，例如 TiKV 的 Store 状态变为 Up，TiDB 的 Member 状态为 healthy，满足这样的条件的 Pod 才被视为升级成功，然后调小 Partition 的值进行下一 Pod 的更新。

### 新 Statefulset 的加工(二): 扩缩容

`m.scaler.Scale` 函数负责扩缩容相关操作，主要是更新 Statefulset 中组件的 Replicas。Scale 遵循逐个扩缩容的原则，每次扩缩容的跨度为 1。Scale 函数会将 TiDBCluster 中指定的组件 Replicas 数，如 `tc.Spec.PD.Replicas`，与现有比较，先判断是扩容需求还是缩容需求，然后完成一个步长的扩缩容的操作，再进入下一次组件 Reconcile，通过多次 Reconcile 完成所有的扩缩容需求。在扩缩容的过程中，会涉及到 PD 转移 Leader，TiKV 删除 Store 等使用 PD API 的操作，组件 Reconcile 过程中会使用 PD API 完成上述操作，并且判断操作是否成功，再逐步进行下一次扩缩容。

### 新 Statefulset 的加工(三): Failover

`m.failover.Failover` 函数负责容灾相关的操作，包括发现和记录灾难状态，恢复灾难状态等，在部署 TiDB Operator 时配置打开 AutoFailover 后，当发现有 Failure 的组件时记录相关信息到 FailureStores 或者 FailureMembers 这样的状态存储的键值，并启动一个新的组件 Pod 用于承担这个 Pod 的工作负载。当原 Pod 恢复工作后，通过修改 Statefulset 的 Replicas 数量，将用于容灾时分担工作负载的 Pod 进行缩容操作。但是在 TiKV/TiFlash 的容灾逻辑中，自动缩容容灾过程中的 Pod 不是默认操作，需要设置 `spec.tikv.recoverFailover: true` 才会对新启动的 Pod 缩容。

### 使用新 Statefulset 进行更新

在同步 Statefulset 的最后阶段，已经完成了新 Statefulset 的生成，这时候会进入 UpdateStatefulSet 函数，这一函数中主要比对新的 Statefulset 和现有 StatefulSet 差异，如果不一致，则进行 Statefulset 的实际更新。另外，这一函数还需要检查是否有没有被管理的 Statefulset，这部分主要是旧版本使用 Helm Chart 部署的 TiDB，需要将这些 Statefulset 纳入 TiDB Operator 的管理，给他们添加依赖标记。

完成上述操作后，TiDBCluster CR 的 Status 更新到最新，相关 Service，ConfigMap 被创建，生成了新的 Statefulset，并且满足了滚动更新，扩缩容，Failover 的工作。组件的 Reconcile 周而复始，监控着组件的生命周期状态，响应生命周期状态改变和用户输入改变，使得集群在符合预期的状态下正常运行。

## 其他生命周期维护工作

除了 TiDB 主要组件的视角之外，还有一些运维操作被编排到了下面的函数实现中，他们分别负责了以下工作：

1. Discovery，用于 PD 启动参数的配置和 TiDB Dashboard Proxy，Discovery 的存在，可以提供一些动态信息供组件索取，避免了修改 ConfigMap 导致 Pod 滚动更新。

2. Reclaim Policy Manager，用于同步 `tc.Spec.PVReclaimPolicy` 的配置，默认配置下会将PV 的 Reclaim Policy 设置为 Retain，降低数据丢失的风险。

3. Orphan Pod Cleaner，用于在 PVC 创建失败的时候清除 Pod，让 Statefulset Controller 重试 Pod 和对应 PVC 的创建。

4. PVC Cleaner 用于 PVC 相关资源清理，清理被标记可以删除的 PVC。

5. PVC Resizer 用于 PVC 的扩容，在云上使用时可以通过修改 TidbCluster 中的 Storage 相关配置修改 PVC 的大小。

6. Meta Manager 用于同步 StoreIDLabel，MemberIDLabel，NamespaceLabel 等信息到 Pod，PVC，PV 的 label 上。

7. TiDBCluster Status Manager 用于同步 TidbMonitor 和 TiDB Dashboard 相关信息。

## 小结

这篇文章介绍了 TiDBCluster 组件的控制循环逻辑的设计，试图让大家了解，当 TiDBCluster Controller 循环触发各个组件的 Reconcile 函数时，组件 Reconcile 函数是按照怎样的流程巡检组件的相关资源，用户预期的状态，是如何通过 Reconcile 函数，变成实际运行的组件。TiDB Operator 中的控制循环都大致符合以上的设计逻辑，在后面的文章中，我们会具体介绍每个组件是如何套用上面的设计逻辑，实现组件的生命周期管理。

如果有什么好的想法，欢迎通过 [#sig-k8s](https://slack.tidb.io/invite?team=tidb-community&channel=sig-k8s&ref=pingcap-tidb-operator) 或 [pingcap/tidb-operator](https://github.com/pingcap/tidb-operator) 参与 TiDB Operator 社区交流。
