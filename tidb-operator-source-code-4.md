---
title: TiDB Operator 源码阅读 (四) 组件的控制循环
author: ['陈逸文']
date: 2021-06-29
summary: 本篇文章将将以 PD 为例详细介绍组件生命周期管理的实现过程和相关代码，并且以 PD 的介绍为基础，介绍其他组件的部分差异。
tags: ['TiDB Operator']
---

[上篇文章](https://pingcap.com/blog-cn/tidb-operator-source-code-3/)中，我们介绍了 TiDB Operator 的组件生命周期管理的编排，以 TiDBCluster Controller 为例介绍 Controller Manager 的实现。TiDBCluster Controller 负责了 TiDB 主要组件的生命周期管理，TiDB 各个组件的 Member Manager 封装了对应具体的生命周期管理逻辑。在上篇文章中，我们描述了一个抽象的组件生命周期管理的实现，本文中，我们将以 PD 为例详细介绍组件生命周期管理的实现过程和相关代码，并且以 PD 的介绍为基础，介绍其他组件的部分差异。

## PD 的生命周期管理

PD 生命周期管理的主要逻辑在 PD Member Manager 下维护，主要代码在 `pkg/manager/member/pd_member_manager.go` 文件中，扩缩容、升级、故障转移的逻辑分别封装在 PD Scaler、PD Upgrader、PD Failover 中，分别位于 `pd_scaler.go`、`pd_upgrader.go`、`pd_failover.go` 文件中。

按照前文描述，组件的生命周期管理主要需要完成以下过程：

1. 同步 Service；

2. 进入 StatefulSet 同步过程；

3. 同步 Status；

4. 同步 ConfigMap；

5. 处理滚动更新；

6. 处理扩容与缩容；

7. 处理故障转移；

8. 最终完成 StatefulSet 同步过程。

其中，同步 StatefulSet 过程为 PD 组件生命周期管理的主要逻辑，其他同步过程，诸如同步 Status、同步 ConfigMap、滚动更新、扩容与缩容、故障转移等任务，被分别定义为了一个个子函数，在同步 StatefulSet 的过程中被调用，这些子任务的实现会在介绍完 StatefulSet 的同步过程之后详细介绍。

### 同步 StatefulSet

1. 使用 Stetefulset Lister 获取 PD 现有的 StatefulSet：

```
oldPDSetTmp, err := m.deps.StatefulSetLister.StatefulSets(ns).Get(controller.PDMemberName(tcName))
if err != nil && !errors.IsNotFound(err) {
    return fmt.Errorf("syncPDStatefulSetForTidbCluster: fail to get sts %s for cluster %s/%s, error: %s", controller.PDMemberName(tcName), ns, tcName, err)
}
setNotExist := errors.IsNotFound(err)
 
oldPDSet := oldPDSetTmp.DeepCopy()
```

2. 使用 `m.syncTidbClusterStatus(tc, oldPDSet)` 获取最新状态。

```
if err := m.syncTidbClusterStatus(tc, oldPDSet); err != nil {
    klog.Errorf("failed to sync TidbCluster: [%s/%s]'s status, error: %v", ns, tcName, err)
}
```

3. 检查 TidbCluster 是否处于 Paused 状态，如果是，则停止接下来的 Reconcile 过程。

```	
if tc.Spec.Paused {
    klog.V(4).Infof("tidb cluster %s/%s is paused, skip syncing for pd statefulset", tc.GetNamespace(), tc.GetName())
    return nil
}
```

4. 根据最新的 `tc.Spec`，对 ConfigMap 进行同步。

```
cm, err := m.syncPDConfigMap(tc, oldPDSet)
```

5. 根据最新的 `tc.Spec`、`tc.Status` 以及上一步获取到的 ConfigMap，生成最新的 StatefulSet 的模板。

```
newPDSet, err := getNewPDSetForTidbCluster(tc, cm)
```

6. 如果 PD 的 StatefulSet 还未创建，那么在这一轮同步中会先创建 PD 的 StatefulSet。

```
if setNotExist {
    if err := SetStatefulSetLastAppliedConfigAnnotation(newPDSet); err != nil {
        return err
    }
    if err := m.deps.StatefulSetControl.CreateStatefulSet(tc, newPDSet); err != nil {
        return err
    }
    tc.Status.PD.StatefulSet = &apps.StatefulSetStatus{}
    return controller.RequeueErrorf("TidbCluster: [%s/%s], waiting for PD cluster running", ns, tcName)
}
```

7. 如果用户使用 Annotation 配置了强制升级，那么会在这一步直接设置 StatefulSet 进行滚动更新，用于有些场景下同步循环被阻塞一直无法更新的情况。

```
if !tc.Status.PD.Synced && NeedForceUpgrade(tc.Annotations) {
    tc.Status.PD.Phase = v1alpha1.UpgradePhase
    setUpgradePartition(newPDSet, 0)
    errSTS := UpdateStatefulSet(m.deps.StatefulSetControl, tc, newPDSet, oldPDSet)
    return controller.RequeueErrorf("tidbcluster: [%s/%s]'s pd needs force upgrade, %v", ns, tcName, errSTS)
}
```

8. 处理 Scale ，调用 `pd_scaler.go` 内实现的扩缩容逻辑。

```
if err := m.scaler.Scale(tc, oldPDSet, newPDSet); err != nil {
    return err
}
```

9. 处理 Failover，调用 `pd_failover.go` 内实现的故障转移逻辑，先检查是否需要 Recover，再检查是否所有 Pod 都正常启动、是否所有 Members 状态都是健康的，再决定是否进入 Failover 逻辑。

```
if m.deps.CLIConfig.AutoFailover {
    if m.shouldRecover(tc) {
        m.failover.Recover(tc)
    } else if tc.PDAllPodsStarted() && !tc.PDAllMembersReady() || tc.PDAutoFailovering() {
        if err := m.failover.Failover(tc); err != nil {
            return err
        }
    }
}
```

10. 处理 Upgrade，调用 `pd_upgrader.go` 内实现的更新逻辑，当前面生成的新 PD StatefulSet 和 Kubernetes 内现有的 PD StatefulSet 不一致，或者 StatefulSet 一致但 `tc.Status.PD.Phase` 里记录的状态是更新状态，都会进入到 Upgrader 里面去处理滚动更新相关逻辑。

```
if !templateEqual(newPDSet, oldPDSet) || tc.Status.PD.Phase == v1alpha1.UpgradePhase {
    if err := m.upgrader.Upgrade(tc, oldPDSet, newPDSet); err != nil {
        return err
    }
}
```

11. 最后实现 PD StatefulSet 的同步，将新的 StatefulSet 更新到 Kubernetes 集群内。

### 同步 Service

PD 使用了 Service 和 Headless Service 两种 Service，由 `syncPDServiceForTidbCluster` 和 `syncPDHeadlessServiceForTidbCluster` 两个函数管理。

Service 地址一般用于 TiKV、TiDB、TiFlash 配置的 PD Endpoint，例如 TiDB 的启动参数如下，其中 TiDB 使用的 `--path=${CLUSTER_NAME}-pd:2379` 就是 PD 的 Service 地址：

```
ARGS="--store=tikv \
--advertise-address=${POD_NAME}.${HEADLESS_SERVICE_NAME}.${NAMESPACE}.svc \
--path=${CLUSTER_NAME}-pd:2379 \
```

Headless Service 可以为每个 Pod 提供唯一网络标识，例如 PD 的启动参数如下所示，当 PD 通过以下参数启动时，该 PD Pod 在 PD Members 中注册的该 Pod 的 Endpoint 为 `"${POD_NAME}.${PEER_SERVICE_NAME}.${NAMESPACE}.svc"`。

```
domain="${POD_NAME}.${PEER_SERVICE_NAME}.${NAMESPACE}.svc"
ARGS="--data-dir=/var/lib/pd \
--name=${POD_NAME} \
--peer-urls=http://0.0.0.0:2380 \
--advertise-peer-urls=http://${domain}:2380 \
--client-urls=http://0.0.0.0:2379 \
--advertise-client-urls=http://${domain}:2379 \
--config=/etc/pd/pd.toml \
"
```

### 同步 ConfigMap

PD 使用 ConfigMap 管理配置和启动脚本，`syncPDConfigMap` 函数调用 `getPDConfigMap` 获得最新的 ConfigMap，然后将最新的 ConfigMap 更新到 Kubernetes 集群中。ConfigMap 需要处理以下任务：

1. 获取 PD Config，用于后面同步。为了兼容 TiDB Operator 1.0 版本使用 Helm 维护 ConfigMap 的情况，当 config 对象为空时，不同步 ConfigMap。

```
config := tc.Spec.PD.Config
if config == nil {
    return nil, nil
}
```

2. 修改配置中 TLS 有关配置，其中 4.0 以下不支持 Dashboard，因此 4.0 以下 PD 版本不用设置 Dashboard 证书。
 
```
// override CA if tls enabled
if tc.IsTLSClusterEnabled() {
    config.Set("security.cacert-path", path.Join(pdClusterCertPath, tlsSecretRootCAKey))
    config.Set("security.cert-path", path.Join(pdClusterCertPath, corev1.TLSCertKey))
    config.Set("security.key-path", path.Join(pdClusterCertPath, corev1.TLSPrivateKeyKey))
}
// Versions below v4.0 do not support Dashboard
if tc.Spec.TiDB != nil && tc.Spec.TiDB.IsTLSClientEnabled() && !tc.SkipTLSWhenConnectTiDB() && clusterVersionGE4 {
    config.Set("dashboard.tidb-cacert-path", path.Join(tidbClientCertPath, tlsSecretRootCAKey))
    config.Set("dashboard.tidb-cert-path", path.Join(tidbClientCertPath, corev1.TLSCertKey))
    config.Set("dashboard.tidb-key-path", path.Join(tidbClientCertPath, corev1.TLSPrivateKeyKey))
}
```

3. 将 Config 转换成 TOML 格式用于 PD 使用。

```
confText, err := config.MarshalTOML()
```

4. 使用 `RenderPDStartScript` 生成 PD 启动脚本，其中 PD 的启动脚本模板在 pkg/manager/member/template.go 中的 `pdStartScriptTpl` 变量中。PD 的启动脚本是一段 Bash 脚本，根据模板渲染的目的是将一些 TidbCluster 对象设置的变量和 Annotation 插入到启动脚本中，用于 PD 正常启动以及 debug 模式。

5. 将上文生成的 PD 配置以及 PD 启动脚本，组装成 Kubernetes 的 ConfigMap 对象，返回给 syncPDConfigMap 函数。

```
cm := &corev1.ConfigMap{
    ObjectMeta: metav1.ObjectMeta{
        Name:            controller.PDMemberName(tc.Name),
        Namespace:       tc.Namespace,
        Labels:          pdLabel,
        OwnerReferences: []metav1.OwnerReference{controller.GetOwnerRef(tc)},
    },
    Data: map[string]string{
        "config-file":    string(confText),
        "startup-script": startScript,
    },
}
```

### 扩容与缩容

扩缩容实现在 `pkg/manager/member/pd_scaler.go` 文件中，用于处理 PD 扩缩容的需求。在 StatefulSet 中，会调用 Scale 函数进入扩缩容的逻辑。扩容缩容都是通过设置 StatefulSet 副本数量实施，实际扩缩容之前需要完成一些前置操作，例如缩容时需要主动迁移 Leader、下线节点、为 PVC 添加延时删除的 Annotation、扩容时自动删除之前保留的 PVC。在完成前置操作后再调整 StatefulSet 副本数量，减少扩缩容操作对集群的影响。这些前置操作也可以根据业务需要进行拓展。

```
func (s *pdScaler) Scale(meta metav1.Object, oldSet *apps.StatefulSet, newSet *apps.StatefulSet) error {
    scaling, _, _, _ := scaleOne(oldSet, newSet)
    if scaling > 0 {
        return s.ScaleOut(meta, oldSet, newSet)
    } else if scaling < 0 {
        return s.ScaleIn(meta, oldSet, newSet)
    }
    return s.SyncAutoScalerAnn(meta, oldSet)
}
```

这个 Scale 函数相当于一个路由，根据扩缩容的方向、步长、距离决定调用方案。目前 PD 扩缩容每次只会扩缩容一个节点，步长为 1，方向根据 scaling 变量正负性决定，方案由 ScaleIn 和 ScaleOut 两个函数实现。

对于 PD 缩容，为了不影响集群性能，缩容过程中需要主动完成 Leader 迁移工作。否则缩容节点为 Leader 节点时，在 Leader 节点下线后剩余节点才会在没有 Leader 的情况下被动开始选举 Leader，影响集群性能。主动迁移 Leader 时，只需要将 Leader 迁移到序号最小的节点，保证 PD Leader 迁移的次数只有一次。

首先获取 PD Client， 获取 PD Leader：

```
pdClient := controller.GetPDClient(s.deps.PDControl, tc)
leader, err := pdClient.GetPDLeader()
```

当 Leader Name 等于该节点 Name 的时候，执行 transferLeader 操作。如果节点数量为 1，此时没有足够 PD 节点完成 transferLeader 的操作，跳过该操作。

Leader 迁移完成后，ScaleIn 函数会调用 PD 的 `DeleteMember` API 从 PD 成员中删除该节点，实现该节点的下线过程，最后调用 `setReplicasAndDeleteSlots` 调整 StatefulSet 的副本数量完成缩容。

对于 PD 扩容，此时缩容为了数据可靠性留下的 PVC，在扩容时需要删掉，防止使用旧数据，因此在扩容前会调用 `deleteDeferDeletingPVC` 删掉延迟删除的 PVC。删除之后，调整 StatefulSet 的副本数量扩容即可。

对于 PD 的扩缩容，主要是通过设置 StatefulSet 副本数量完成扩缩容，因此在支持 Advanced StatefulSet 时，在计算副本数量需要考虑留空 slots 的存在。

### 滚动更新

PD 升级在 `pkg/manager/member/pd_upgrader.go` 中实现，主要手段是使用 StatefulSet 的 UpdateStrategy 实现滚动更新。PD Upgrader 中会在调整 StatefulSet UpgradeStrategy 过程中插入 PD 的一些前置操作，减少升级操作对 PD 集群的影响。对于具体如何控制 StatefulSet UpgradeStrategy，可以参考[上篇文章](https://pingcap.com/blog-cn/tidb-operator-source-code-3/)。

在开始升级前，需要完成以下状态检查：

1. 检查有无其他操作正在进行，主要是检查 TiCDC、TiFlash 是否处于升级状态，PD 是否处于扩缩容状态：

```
if tc.Status.TiCDC.Phase == v1alpha1.UpgradePhase ||
        tc.Status.TiFlash.Phase == v1alpha1.UpgradePhase ||
        tc.PDScaling()
```

2. 在同步 StatefulSet 部分提到，进入 Upgrader 有两种条件，一是 newSet 和 oldSet Template Spec 不一致，这一情况发生在更新开始时，此时返回 nil，直接更新 StatefulSet 即可，不需要执行下面逐个 Pod 检查的操作。如果是根据 `tc.Status.PD.Phase == v1alpha1.UpgradePhase` 条件进入的 Upgrader，则 `newSet` 和 `oldSet Template Spec` 一致，此时需要继续执行下面的检查。

``` 
if !templateEqual(newSet, oldSet) {
    return nil
}
```

3. 比较 `tc.Status.PD.StatefulSet.UpdateRevision` 和 `tc.Status.PD.StatefulSet.CurrentRevision` 来获取滚动更新状态，若两者相等说明滚动更新操作完成，可以退出滚动更新过程。

```
if tc.Status.PD.StatefulSet.UpdateRevision == tc.Status.PD.StatefulSet.CurrentRevision
```

4. 检查 `StatefulSet` 的 `UpdateStrategy` 是否被手动修改，如果手动修改则沿用相应策略。

```
if oldSet.Spec.UpdateStrategy.Type == apps.OnDeleteStatefulSetStrategyType || oldSet.Spec.UpdateStrategy.RollingUpdate == nil {
        newSet.Spec.UpdateStrategy = oldSet.Spec.UpdateStrategy
        klog.Warningf("tidbcluster: [%s/%s] pd statefulset %s UpdateStrategy has been modified manually", ns, tcName, oldSet.GetName())
        return nil
    }
```

完成整体检查后，开始对每个 Pod 进行处理，进行滚动更新操作：

1. 检查 PD Pod 是否已经更新完毕，通过检查 Pod Label 中的 controller-revision-hash 的值和 `StatefulSet` 的 `UpdateRevision` 比较，判断该 Pod 是已经升级完成的 Pod 还是未处理的 Pod。对于已进行升级完成的 Pod，则检查该 Pod 对应的 PD Member 是否变为健康状态，若不是则返回错误，等待下一次同步继续检查状态，若达到健康状态则开始处理下一个 Pod。

```
revision, exist := pod.Labels[apps.ControllerRevisionHashLabelKey]
        if !exist {
            return controller.RequeueErrorf("tidbcluster: [%s/%s]'s pd pod: [%s] has no label: %s", ns, tcName, podName, apps.ControllerRevisionHashLabelKey)
        }
 
        if revision == tc.Status.PD.StatefulSet.UpdateRevision {
            if member, exist := tc.Status.PD.Members[PdName(tc.Name, i, tc.Namespace, tc.Spec.ClusterDomain)]; !exist || !member.Health {
                return controller.RequeueErrorf("tidbcluster: [%s/%s]'s pd upgraded pod: [%s] is not ready", ns, tcName, podName)
            }
            continue
        }
```

2. 对于 ```Pod revision != tc.Status.PD.StatefulSet.UpdateRevision``` 的 Pod，说明该 Pod 还未执行滚动更新，调用  upgradePDPod 函数处理该 Pod。与缩容逻辑一样，当处理到 PD Leader Pod 时，会执行一段主动迁移 Leader 的操作，然后才会执行更新该 Pod 的操作。

```
if tc.Status.PD.Leader.Name == upgradePdName || tc.Status.PD.Leader.Name == upgradePodName {
    var targetName string
    targetOrdinal := helper.GetMaxPodOrdinal(*newSet.Spec.Replicas, newSet)
    if ordinal == targetOrdinal {
        targetOrdinal = helper.GetMinPodOrdinal(*newSet.Spec.Replicas, newSet)
    }
    targetName = PdName(tcName, targetOrdinal, tc.Namespace, tc.Spec.ClusterDomain)
    if _, exist := tc.Status.PD.Members[targetName]; !exist {
        targetName = PdPodName(tcName, targetOrdinal)
    }
    
    if len(targetName) > 0 {
        err := u.transferPDLeaderTo(tc, targetName)
        if err != nil {
            klog.Errorf("pd upgrader: failed to transfer pd leader to: %s, %v", targetName, err)
            return err
        }
        klog.Infof("pd upgrader: transfer pd leader to: %s successfully", targetName)
        return controller.RequeueErrorf("tidbcluster: [%s/%s]'s pd member: [%s] is transferring leader to pd member: [%s]", ns, tcName, upgradePdName, targetName)
    }
}
setUpgradePartition(newSet, ordinal)
```

### 故障转移

PD 故障转移在 `pd_failover.go` 中实现，与其他组件 Failover 逻辑不同的是，PD 会通过主动删除故障 Pod 来修复问题。PD Failover 之前会做相应检查，当 PD 集群不可用，即过半 PD 节点不健康，此时重建 PD 节点并不会使集群恢复，因此不会进行 Failover 工作。

1. 遍历从 PD Client 获取到的 PD Members 健康状态，当 PD 成员健康状态为 Unhealthy 且LastTransitionTime 距现在时间超过 failoverDeadline 时间，则标记为不健康成员，接下来的操作会将不健康的 PD 成员相关的 Pod 信息、PVC 信息记录到 `tc.Status.PD.FailureMembers` 中。

```
for pdName, pdMember := range tc.Status.PD.Members {
    podName := strings.Split(pdName, ".")[0]
 
    failoverDeadline := pdMember.LastTransitionTime.Add(f.deps.CLIConfig.PDFailoverPeriod)
    _, exist := tc.Status.PD.FailureMembers[pdName]
 
    if pdMember.Health || time.Now().Before(failoverDeadline) || exist {
        continue
    }
 
    pod, _ := f.deps.PodLister.Pods(ns).Get(podName)
 
    pvcs, _ := util.ResolvePVCFromPod(pod, f.deps.PVCLister)
 
    f.deps.Recorder.Eventf(tc, apiv1.EventTypeWarning, "PDMemberUnhealthy", "%s/%s(%s) is unhealthy", ns, podName, pdMember.ID)
 
    pvcUIDSet := make(map[types.UID]struct{})
    for _, pvc := range pvcs {
        pvcUIDSet[pvc.UID] = struct{}{}
    }
    tc.Status.PD.FailureMembers[pdName] = v1alpha1.PDFailureMember{
        PodName:       podName,
        MemberID:      pdMember.ID,
        PVCUIDSet:     pvcUIDSet,
        MemberDeleted: false,
        CreatedAt:     metav1.Now(),
    }
    return controller.RequeueErrorf("marking Pod: %s/%s pd member: %s as failure", ns, podName, pdMember.Name)
}
```

2. 调用 tryToDeleteAFailureMember 函数处理 FailureMemebrs，遍历 FailureMembers，当遇到 MemberDeleted 为 False 的成员，调用 PD Client 删除该 PD 成员，尝试恢复 Pod。

```
func (f *pdFailover) tryToDeleteAFailureMember(tc *v1alpha1.TidbCluster) error {
    ns := tc.GetNamespace()
    tcName := tc.GetName()
    var failureMember *v1alpha1.PDFailureMember
    var failurePodName string
    var failurePDName string
 
    for pdName, pdMember := range tc.Status.PD.FailureMembers {
        if !pdMember.MemberDeleted {
            failureMember = &pdMember
            failurePodName = strings.Split(pdName, ".")[0]
            failurePDName = pdName
            break
        }
    }
    if failureMember == nil {
        klog.Infof("No PD FailureMembers to delete for tc %s/%s", ns, tcName)
        return nil
    }
 
    memberID, err := strconv.ParseUint(failureMember.MemberID, 10, 64)
    if err != nil {
        return err
    }
 
    if err := controller.GetPDClient(f.deps.PDControl, tc).DeleteMemberByID(memberID); err != nil {
        klog.Errorf("pd failover[tryToDeleteAFailureMember]: failed to delete member %s/%s(%d), error: %v", ns, failurePodName, memberID, err)
        return err
    }
    klog.Infof("pd failover[tryToDeleteAFailureMember]: delete member %s/%s(%d) successfully", ns, failurePodName, memberID)
  ...
```

删除该故障 Pod，

```
pod, err := f.deps.PodLister.Pods(ns).Get(failurePodName)
    if err != nil && !errors.IsNotFound(err) {
        return fmt.Errorf("pd failover[tryToDeleteAFailureMember]: failed to get pod %s/%s for tc %s/%s, error: %s", ns, failurePodName, ns, tcName, err)
    }
    if pod != nil {
        if pod.DeletionTimestamp == nil {
            if err := f.deps.PodControl.DeletePod(tc, pod); err != nil {
                return err
            }
        }
    } else {
        klog.Infof("pd failover[tryToDeleteAFailureMember]: failure pod %s/%s not found, skip", ns, failurePodName)
    }
```

删除 PVC，

```
for _, pvc := range pvcs {
        _, pvcUIDExist := failureMember.PVCUIDSet[pvc.GetUID()]
        // for backward compatibility, if there exists failureMembers and user upgrades operator to newer version
        // there will be failure member structures with PVCUID set from api server, we should handle this as pvcUIDExist == true
        if pvc.GetUID() == failureMember.PVCUID {
            pvcUIDExist = true
        }
        if pvc.DeletionTimestamp == nil && pvcUIDExist {
            if err := f.deps.PVCControl.DeletePVC(tc, pvc); err != nil {
                klog.Errorf("pd failover[tryToDeleteAFailureMember]: failed to delete PVC: %s/%s, error: %s", ns, pvc.Name, err)
                return err
            }
            klog.Infof("pd failover[tryToDeleteAFailureMember]: delete PVC %s/%s successfully", ns, pvc.Name)
        }
    }
```

标记 `tc.Status.PD.FailureMembers` 中的状态为 Deleted。

```
setMemberDeleted(tc, failurePDName)
```

3. PD StatefulSet 的 replicas 副本数量由 `tc.PDStsDesiredReplicas()` 获得，StatefulSet 副本数量会加上已经被删除的 FailureMembers 数量，此时同步 StatefulSet 过程中会调用扩容 StatefulSet 的逻辑增加一个用于故障转移的 PD Pod。

```
func (tc *TidbCluster) GetPDDeletedFailureReplicas() int32 {
    var deletedReplicas int32 = 0
    for _, failureMember := range tc.Status.PD.FailureMembers {
        if failureMember.MemberDeleted {
            deletedReplicas++
        }
    }
    return deletedReplicas
}
 
func (tc *TidbCluster) PDStsDesiredReplicas() int32 {
    return tc.Spec.PD.Replicas + tc.GetPDDeletedFailureReplicas()
}
```

## 其他组件的生命周期管理

在前面我们通过 PD 详细介绍了组件的生命周期管理的代码实现，其他组件，包括 TiKV、TiFlash、TiDB、Pump、TiCDC，与 PD 的生命周期管理较为类似，在此不再赘述，在下面部分我们着重强调和 PD 代码实现的差异。

### TiKV/TiFlash 的生命周期管理

TiKV 与 TiFlash 生命周期管理比较类似，理解 TiKV 的代码实现即可。 TiKV 的生命周期管理与 PD 生命周期管理差异如下：

1. 同步 StatefulSet 过程中，TiKV Member Manager 需要通过 setStoreLabelsForTiKV 设置 TiKV Store Label。setStoreLabelsForTiKV 函数中实现了将 Node 上的 Label 通过 PD Client 的 SetStoreLabels 接口为 TiKV store 设置 Label。

```
for _, store := range storesInfo.Stores {
    nodeName := pod.Spec.NodeName
    ls, _ := getNodeLabels(m.deps.NodeLister, nodeName, storeLabels)
 
    if !m.storeLabelsEqualNodeLabels(store.Store.Labels, ls) {
        set, err := pdCli.SetStoreLabels(store.Store.Id, ls)
        if err != nil {
            continue
        }
        if set {
            setCount++
            klog.Infof("pod: [%s/%s] set labels: %v successfully", ns, podName, ls)
        }
    }
}
```

2. 同步 Status 方面，TiKV Member Manager 会调用 PD Client 的 GetStores 函数从 PD 获得 TiKV Stores 信息，并对得到的 Stores 信息进行分类，用于后续同步。这一过程类似 PD 的 Status 同步过程中对 GetMembers 接口的调用并对 PD Members 信息进行分析和记录的过程。

3. 同步 Service 方面，TiKV Member Manager 只创建 Headless Service，用于 TiKV Pod DNS 解析。

4. 同步 ConfigMap 方面，TiKV Member Manager 与 PD Member Manager 类似，相关脚本模版实现在 templates.go 文件中，通过调用 RenderTiKVStartScript 生成 TiKV 的启动脚本，调用 transformTiKVConfigMap 获得 TiKV 的配置文件。

5. 扩容与缩容方面，与 PD 生命周期管理缩容前中主动迁移 Leader 节点类似，TiKV 生命周期管理中需要安全下线 TiKV Stores，TiKV Member Manager 调用 PD Client 的 DeleteStore 删除该 Pod 上对应的 Store。

6. 滚动更新方面，TiKV 需要在滚动更新 Pod 之前，需要确保 Pod 上没有 Region Leader。在开始滚动更新前，TiKV Upgrader 中会通过 Pod Annotation 上有没有 EvictLeaderBeginTime 判断是否对该 Pod 进行过 EvictLeader 操作。如果没有，则调用 调用 PD Client 中的 BeginEvictLeader 函数，为 TiKV store 添加 evict leader scheduler 驱逐该 TiKV Store 上的 Region Leader。

```
_, evicting := upgradePod.Annotations[EvictLeaderBeginTime]
if !evicting {
    return u.beginEvictLeader(tc, storeID, upgradePod)
}
```
 
在 readyToUpgrade 函数中，当 Region Leader 为零，或者转移 Region Leader时间超过了 `tc.Spec.TiKV.EvictLeaderTimeout` 设置的时间，则更新 StatefulSet UpgradeStrategy 中的 partition 配置，触发 Pod 升级。当 Pod 完成升级之后，调用 endEvictLeaderbyStoreID 结束 evictLeader 操作。

7. 故障转移方面，TiKV Member Manager 在状态同步时，会记录最后一次 Store 状态变更的时间。

```
status.LastTransitionTime = metav1.Now()
if exist && status.State == oldStore.State {
    status.LastTransitionTime = oldStore.LastTransitionTime
}
```

当 Store 状态为 `v1alpha1.TiKVStateDown`，并且根据 LastTransitionTime 保持 Down 的状态的时间超过了配置中设置的 Failover 时间限制，则将该 TiKV Pod 加入到 FailureStores 中，

```
if store.State == v1alpha1.TiKVStateDown && time.Now().After(deadline) && !exist {
    if tc.Status.TiKV.FailureStores == nil {
        tc.Status.TiKV.FailureStores = map[string]v1alpha1.TiKVFailureStore{}
    }
    if tc.Spec.TiKV.MaxFailoverCount != nil && *tc.Spec.TiKV.MaxFailoverCount > 0 {
        maxFailoverCount := *tc.Spec.TiKV.MaxFailoverCount
        if len(tc.Status.TiKV.FailureStores) >= int(maxFailoverCount) {
            klog.Warningf("%s/%s failure stores count reached the limit: %d", ns, tcName, tc.Spec.TiKV.MaxFailoverCount)
            return nil
        }
        tc.Status.TiKV.FailureStores[storeID] = v1alpha1.TiKVFailureStore{
            PodName:   podName,
            StoreID:   store.ID,
            CreatedAt: metav1.Now(),
        }
        msg := fmt.Sprintf("store[%s] is Down", store.ID)
        f.deps.Recorder.Event(tc, corev1.EventTypeWarning, unHealthEventReason, fmt.Sprintf(unHealthEventMsgPattern, "tikv", podName, msg))
    }
}
```

同步 TiKV StatefulSet 过程中，replicas 数量会加上 FailureStores 的数量，触发扩容逻辑，完成 failover 过程。

```
func (tc *TidbCluster) TiKVStsDesiredReplicas() int32 {
   return tc.Spec.TiKV.Replicas + int32(len(tc.Status.TiKV.FailureStores))
}
```

### TiDB/TiCDC/Pump 的生命周期管理

TiDB、TiCDC、Pump 的生命周期管理比较类似，与其他组件相比，主要需要控制滚动更新时成员状态为健康状态时才允许继续滚动更新过程。在扩容与缩容过程中，额外需要考虑的是 PVC 的使用，与 PD 的 PVC 使用方法类似，需要在缩容时添加 deferDeleting 的设计保证数据安全、在扩容时移除该 PVC。故障转移方面，目前 TiDB 已经实现故障转移的逻辑，TiCDC 和 Pump 暂时没有故障转移相关的逻辑。

## 小结

这篇文章介绍了 TiDBCluster 组件的控制循环的具体实现，主要结合 PD 组件的上下文信息解释了上篇文章中介绍的通用逻辑设计，然后介绍了其他组件的部分差异。通过这篇文章与上一篇文章，我们了解了 TiDB 主要组件的 Member Manager 部分的设计，了解了 TiDB 生命周期管理过程在 TiDB Operator 的实现。

如果有什么好的想法，欢迎通过 [#sig-k8s](https://slack.tidb.io/invite?team=tidb-community&channel=sig-k8s&ref=pingcap-tidb-operator) 或 [pingcap/tidb-operator](https://github.com/pingcap/tidb-operator) 参与 TiDB Operator 社区交流。
