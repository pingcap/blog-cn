---
title: TiDB Operator 源码阅读 (二) Operator 模式
author: ['陈逸文']
date: 2021-03-19
summary: 本文将从 Operator 模式的视角，介绍 TiDB Operator 的代码组织逻辑，我们将会分析从代码入口到组件的生命周期事件被触发中间的过程。
tags: ['TiDB Operator']
---

在[上一篇文章](https://pingcap.com/blog-cn/tidb-operator-source-code-1/)中我们讨论了 TiDB Operator 的应用场景，了解了 TiDB Operator 可以在 Kubernetes 集群中管理 TiDB 的生命周期。可是，TiDB Operator 的代码是怎样运行起来的？TiDB 组件的生命周期管理的逻辑又是如何编排的呢？在这篇文章中，我们将从 Operator 模式的视角，介绍 TiDB Operator 的代码组织逻辑，我们将会分析从代码入口到组件的生命周期事件被触发中间的过程。

## Operator模式的演化: 从 Controller 模式到 Operator 模式

TiDB Operator 参考了 kube-controller-manager 的设计，了解 Kubernetes 的设计有助于了解 TiDB Operator 的代码逻辑。Kubernetes 内的 Resources 都是通过 Controller 实现生命周期管理的，例如 Namespace、Node、Deployment、Statefulset 等等，这些 Controller 的代码在 kube-controller-manager 中实现并由 kube-controller-manager 启动后调用。

为了支持用户自定义资源的开发需求，Kubernetes 社区基于上面的开发经验，提出了 Operator 模式。Kubernetes 支持通过 CRD（CustomResourceDefinition）来描述自定义资源，从这个定义创建出来的对象叫 CR（CustomResource），开发者实现相应 controller 通过监听 CR 及关联资源的变更，通过比对资源最新状态和期望状态，分步骤或一次性完成运维操作，实现最终资源状态与期望状态一致。通过定义 CRD 和实现对应 controller，无需将代码合并到 Kubernetes 中编译使用， 即可完成一个资源的生命周期管理。

## TiDB Operator 的 Controller Manager

TiDB Operator 使用 tidb-controller-manager 管理各个 CRD 的 controller。从 cmd/controller-manager/main.go 开始，tidb-controller-manager 首先加载了 kubeconfig，用于连接 kube-apiserver，然后使用一系列 NewController 函数，加载了各个 Controller 的初始化函数。

```go
controllers := []Controller{
    tidbcluster.NewController(deps),
    dmcluster.NewController(deps),
    backup.NewController(deps),
    restore.NewController(deps),
    backupschedule.NewController(deps),
    tidbinitializer.NewController(deps),
    tidbmonitor.NewController(deps),
}
```
在加载 controller 的初始化函数过程中，tidb-controller-manager 会接着初始化一系列 informer，这些informer主要用来和 kube-apiserver 交互获取 CRD 和相关资源的变更。以 TiDBCluster 为例，在初始化函数 NewController 中，会初始化 Informer 对象：
 
```go
tidbClusterInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: c.enqueueTidbCluster,
        UpdateFunc: func(old, cur interface{}) {
            c.enqueueTidbCluster(cur)
        },
        DeleteFunc: c.enqueueTidbCluster,
    })
statefulsetInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: c.addStatefulSet,
        UpdateFunc: func(old, cur interface{}) {
            c.updateStatefulSet(old, cur)
        },
        DeleteFunc: c.deleteStatefulSet,
    })
 
```
 
Informer 中添加了处理添加，更新，删除事件的 Eventhandler，把监听到的事件涉及到的 CR 的 Key 加入队列。
 
初始化完成后启动 InformerFactory 并等待 cache 同步完成。
 
```go
informerFactories := []InformerFactory{
            deps.InformerFactory,
            deps.KubeInformerFactory,
            deps.LabelFilterKubeInformerFactory,
        }
        for _, f := range informerFactories {
            f.Start(ctx.Done())
            for v, synced := range f.WaitForCacheSync(wait.NeverStop) {
                if !synced {
                    klog.Fatalf("error syncing informer for %v", v)
                }
            }
        }
```
 
随后 tidb-controller-manager 会调用各个 Controller 的 Run 函数，开始循环执行 controller 的内部逻辑。
 
```go
// Start syncLoop for all controllers
for _,controller := range controllers {
    c := controller
    go wait.Forever(func() { c.Run(cliCfg.Workers,ctx.Done()) },cliCfg.WaitDuration)
}
```
 
以 TiDBCluster Controller 为例，Run 函数会启动 worker 处理工作队列。
 
```go
// Run runs the tidbcluster controller.
func (c *Controller) Run(workers int, stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    defer c.queue.ShutDown()
 
    klog.Info("Starting tidbcluster controller")
    defer klog.Info("Shutting down tidbcluster controller")
 
    for i := 0; i < workers; i++ {
        go wait.Until(c.worker, time.Second, stopCh)
    }
 
    <-stopCh
}
```
 
Worker 会调用 processNextWorkItem 函数，弹出队列的元素，然后调用 sync 函数进行同步：
 
```go
// worker runs a worker goroutine that invokes processNextWorkItem until the the controller's queue is closed
func (c *Controller) worker() {
    for c.processNextWorkItem() {
    }
}
 
// processNextWorkItem dequeues items, processes them, and marks them done. It enforces that the syncHandler is never
// invoked concurrently with the same key.
func (c *Controller) processNextWorkItem() bool {
    key, quit := c.queue.Get()
    if quit {
        return false
    }
    defer c.queue.Done(key)
    if err := c.sync(key.(string)); err != nil {
        if perrors.Find(err, controller.IsRequeueError) != nil {
            klog.Infof("TidbCluster: %v, still need sync: %v, requeuing", key.(string), err)
        } else {
            utilruntime.HandleError(fmt.Errorf("TidbCluster: %v, sync failed %v, requeuing", key.(string), err))
        }
        c.queue.AddRateLimited(key)
    } else {
        c.queue.Forget(key)
    }
    return true
}
```
 
sync 函数会根据 Key 获取对应的 CR 对象，例如这里的 TiDBCluster 对象，然后对这个 TiDBCluster 对象进行同步。
 
```go
// sync syncs the given tidbcluster.
func (c *Controller) sync(key string) error {
    startTime := time.Now()
    defer func() {
        klog.V(4).Infof("Finished syncing TidbCluster %q (%v)", key, time.Since(startTime))
    }()
 
    ns, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }
    tc, err := c.deps.TiDBClusterLister.TidbClusters(ns).Get(name)
    if errors.IsNotFound(err) {
        klog.Infof("TidbCluster has been deleted %v", key)
        return nil
    }
    if err != nil {
        return err
    }
 
    return c.syncTidbCluster(tc.DeepCopy())
}
 
func (c *Controller) syncTidbCluster(tc *v1alpha1.TidbCluster) error {
    return c.control.UpdateTidbCluster(tc)
}
```
 
syncTidbCluster 函数调用 updateTidbCluster 函数，进而调用一系列组件的 sync 函数实现 TiDB 集群管理的相关工作。在 pkg/controller/tidbcluster/tidb_cluster_control.go 的 updateTidbCluster 函数实现中，我们可以看到各个组件的 Sync 函数在这里调用，在相关调用代码附近注释里描述着每个 Sync 函数执行的生命周期操作事件，可以帮助理解每个组件的 Reconcile 需要完成哪些工作，例如 PD 组件:
 
```go
// works that should do to making the pd cluster current state match the desired state:
//   - create or update the pd service
//   - create or update the pd headless service
//   - create the pd statefulset
//   - sync pd cluster status from pd to TidbCluster object
//   - upgrade the pd cluster
//   - scale out/in the pd cluster
//   - failover the pd cluster
if err := c.pdMemberManager.Sync(tc); err != nil {
    return err
}
```
 
我们将在下篇文章中介绍组件的 sync 函数完成了哪些工作，TiDBCluster Controller 是怎样完成各个组件的生命周期管理。

## 小结
通过这篇文章，我们了解到 TiDB Operator 如何从 cmd/controller-manager/main.go 初始化运行到各个 controller 对象的实现，并以 TidbCluster controller 为例介绍了 controller 从初始化到实际工作的过程以及 controller 内部的工作逻辑。通过上面的代码运行逻辑的介绍，我们搞清楚了组件的生命周期控制循环是如何被触发的，问题已经被缩小到如何细化这个控制循环，添加 TiDB 特殊的运维逻辑，使得 TiDB 能在 Kubernetes 上部署和正常运行，完成其他的生命周期操作。

我们介绍了社区对于 Operator 模式的探索和演化。对于一些希望使用 Operator 模式开发资源管理系统的小伙伴，Kubernetes 社区中提供了 Kubebuilder 和 Operator Framework 两个 Controller 脚手架项目。相比于参考 [kubernetes/sample-controller](https://github.com/kubernetes/sample-controller) 进行开发，Operator 脚手架基于 [kubernetes-sigs/controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) 生成 controller 代码，减少了许多重复引入的模板化的代码。开发者只需要专注于完成 CRD 对象的 Reconcile Loop 部分即可，而不需要关心 Reconcile Loop 启动之前的准备工作。

我们将在下一篇文章中讨论如何细化这个控制循环，讨论组件的 Reconcile Loop 的实现。如果有什么好的想法，欢迎通过 [#sig-k8s](https://slack.tidb.io/invite?team=tidb-community&channel=sig-k8s&ref=pingcap-tidb-operator) 或 [pingcap/tidb-operator](https://github.com/pingcap/tidb-operator) 参与 TiDB Operator 社区交流。