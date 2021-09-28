---
title: TiDB Operator 源码阅读 (五) 备份与恢复
author: ['李逸龙']
date: 2021-09-13
summary: 本篇文章主要介绍了 TiDB Operator 提供的备份与恢复功能的实现与设计。本文为本系列文章的第五篇。
tags: ['TiDB Operator']
---
## 前言
备份与恢复是数据库运维场景中非常重要且频繁的操作。运维人员通常需要维护一套脚本来实现自动化定时备份，以确保业务数据的安全，并且能在出现数据损坏需要恢复时，方便快速地执行指定数据集的恢复任务。

对于一个设计良好的备份和恢复操作平台而言，需要能**指定数据源和备份存储目标、定时执行备份任务、维护备份恢复历史**以提供良好的事后审查机制、定时清理老旧的存储以释放不必要的存储空间等。针对以上的需求，TiDB Operator 提供了相应的 **CRD** 来提供管理能力。

通常而言，软件的架构设计与具体实现没有必然的联系，因此本文着重讲**设计和有所抽象的核心逻辑**，而不是着重于可以有多种选择的具体实现细节。
## 控制器
TiDB Operator 通过 **Backup、Restore、BackupSchedule** 等 **CR** 来指定执行备份、恢复和定时备份工作，因此实现了三个对应的控制器来执行对应的控制循环。
为了方便理解，先介绍下用户侧感受到的具体功能示例。当用户需要执行备份任务时，可以定义如下 **yaml** 并提交给 Kubernetes，这里以 Backup 为例，并没有覆盖所有选项。

```
apiVersion: pingcap.com/v1alpha1
kind: Backup
metadata:
  name: demo-backup-gcp
  namespace: test1
spec:
  br:
    cluster: mycluster
  gcs:
    projectId: gcp
    location: us-west2
    bucket: backup
    prefix: test1-demo1
    secretName: gcp-secret
```

在 backup 控制器收到创建 Backup 资源的事件时，便会创建相应的 Job 去完成配置的备份工作，在这里就是备份 test1 命名空间下的 **mycluster** 数据库的数据，备份目标为 **gcp 字段中的配置项指定的 GCP 存储**。

下面分别介绍 backup、restore 和 backupschedule 这三个控制器的主要逻辑。
### backup
backup 控制器管理的是 Backup 资源，会根据 **Backup spec** 中的配置，使用 **br 或 dumpling** 工具执行备份任务，并在用户删除 Backup CR 之后删除对应的备份文件。

跟其他所有的控制器一样，backup 控制器的核心逻辑是一个**控制循环**，监听 Backup 资源的创建、更新和删除事件，并根据 spec 中的配置执行对应的备份操作。因此这里省略最外层通用的控制循环逻辑，主要介绍核心的备份逻辑。
#### 核心逻辑
backup 控制器的核心逻辑在 **pkg/backup/backup/backup_manager.go 文件中的 syncBackupJob 函数内**。由于实际代码中需要处理非常多的 **corner case**，为了方便读者理解，我们在这里把一些细枝末节的逻辑砍掉，只留下核心的代码，因此有部分函数签名可能会有不完全一致的情况，并不影响读者自行阅读源码。只保留核心逻辑后的代码，大致长成这样：
```func (bm *backupManager) syncBackupJob(backup *v1alpha1.Backup) error {
    backuputil.ValidateBackup(backup)
    if err := JobLister.Jobs(ns).Get(backupJobName); err == nil {
        return nil
    } else if !errors.IsNotFound(err) {
        return err
    }
    if backup.Spec.BR == nil {
        // create Job Spec which will use dumpling to do the work
        job = bm.makeExportJob(backup)
    } else {
        // create Job Spec which will use br to do the work
        job = bm.makeBackupJob(backup)
    }
    if err := bm.deps.JobControl.CreateJob(backup, job); err != nil {
        // update Backup.Status with error message
    }
}
```

在上述代码块中，backup 是用户提交的 **Backup yaml** 对应的 **go struct**。首先我们会先使用 ValidateBackup 函数验证 backup 字段的合法性，避免不合法字段的提交。

由于实际的备份任务以 Kubernetes 原生的 Job 形式执行，且控制器的核心要点之一是需要执行多次不影响最终结果，也就是所谓的**幂等性**，因此有可能在这次执行时，已经存在该 Backup 对应的 Job。所以在代码块的第三行，我们会先尝试在 Backup 的命名空间中获取对应的 Job：如果获取到了，说明已经有正在执行的 Job，则直接返回，终止对当前 Backup 对象的处理；如果没获取到，再进一步往下走。

用户在创建 Backup 的时候，如果配置了 br 字段，则代表使用 br 执行备份操作，否则使用 dumpling，因此代码块的第 8-14 行根据这个判断分别执行对应的生成 Job 定义的函数。

第 15 行根据上面返回的 Job 定义，调用 CreateJob 生成实际的 Job 对象，在 Job Pod 中执行真正的备份操作。这部分逻辑会在下面的 backup-manager 一节展开介绍。
#### 生成 Job
实际任务由 **makeExportJob/makeBackupJob** 这两个函数生成，这里以 makeBackupJob 为例，其中最核心的代码经过精简，大致如下所示：

```func (bm *backupManager) makeBackupJob(backup *v1alpha1.Backup) (*batchv1.Job, string, error) {
    tc := bm.deps.TiDBClusterLister.TidbClusters(backupNamespace).Get(backup.Spec.BR.Cluster)
    envVars := backuputil.GenerateTidbPasswordEnv(ns, name, backup, bm)
    envVars = append(envVars, backuputil.GenerateStorageCertEnv(ns, backup, bm))
    args := []string{"backup", fmt.Sprintf("--namespace=%s", ns), fmt.Sprintf("--backupName=%s", name)}
    podSpec := &corev1.PodTemplateSpec{
        Spec: corev1.PodSpec{
            InitContainers: []corev1.Container{{
                Image:   "pingcap/br",
                Command: []string{"/bin/sh", "-c"},
                Args:    []string{fmt.Sprintf("cp /br %s/br; echo 'BR copy finished'", util.BRBinPath)},
            }},
            Containers: []corev1.Container{{
                Image:           "pingcap/tidb-backup-manager",
                Args:            args,
                Env:             util.AppendEnvIfPresent(envVars, "TZ"),
            }},
            Volumes:          volumes,
        },
    }
    job := &batchv1.Job{
        Spec: batchv1.JobSpec{
            Template:     *podSpec,
        },
    }
}
```

大致来讲，上面代码的流程其实很简单。首先根据 backup 信息获取了对应的 **TidbCluster** 资源，然后设置了对应的环境变量和命令行参数。最后构造了使用 **pingcap/tidb-backup-manager** 为镜像的 Job 来执行对应的备份任务。这里使用了 pingcap/br 镜像作为 **init container**，主要是为了把 br 工具 copy 到实际执行任务的容器中去。当这个 Job 被创建之后，实际执行的就是下面章节会讲到的 backup-manager 的逻辑，这里就不展开了。

makeExportJob 函数的核心流程与上述类似，只是最后 backup-manager 会调用 dumpling 而不是 br 工具。
#### 清理备份
backup 控制器除了负责根据用户提交的 Backup 资源创建对应的备份任务，也负责在用户删除 Backup 资源后，删除对应的备份文件，回收存储空间。

在前面章节讲到，backup 控制器会在接收到 Backup 资源创建事件时，生成对应的 Job。实际上与此同时，也给这个 Backup 资源的 **finalizer** 属性中添加了字符串 tidb.pingcap.com/backup-protection，用于标明删除时需要经过特殊处理，而不是直接被删掉。在用户删除这个 Backup 资源时，API server 会给 Backup 设置 DeletionTimestamp 属性，此时 backup 控制器会检查 Backup 的 CleanPolicy，只要用户设置的指不是 Retain，就会创建一个 clean job ，来执行相应的备份文件删除操作，**回收对应的存储空间**。
### restore
恢复是备份的逆向操作，其主要逻辑跟备份类似。首先 restore 控制器会验证用户提交的 Restore 对象字段的**合法性**，保证没有非法字段。然后会根据 Restore 配置选择调用 makeImportJob（使用 lightning）或者 makeRestoreJob（使用 br）来生成对应的恢复 Job。与备份逻辑一样，恢复任务的 Job 中也会使用 backup-manager 作为基础镜像。
### backupschedule
为了保证业务数据的安全，我们通常需要以**定时**的方式**自动地**对重要数据进行备份，backupschedule 控制器就实现了这样的功能。用户可以设置 cron 格式的定时任务，设置跟 Backup 类似的配置，提交给 **API server** 之后，TiDB Operator 就能按照这个配置，定时执行备份操作。为了防止存储资源超限，我们还提供了最长保留多少个备份、多少时间备份的配置，超过这个窗口的过时备份会被清理掉，以释放存储空间。

backupschedule 控制器的核心设计思路是**利用 backup 控制器的已有功能，在外面封装一层定时执行的逻辑抽象**。在需要执行备份任务时，backupschedule 控制器不需要跟 backup 控制器一样，从验证到创建  Job 全部做一遍，而是只需要创建一个 Backup 资源，剩下的事情由 backup 控制器去执行就好了。熟悉 Kubernetes 的读者可能已经发现了，这个设计思路跟 CronJob 与 Job 控制器的关系非常相似，事实上 backupschedule 控制器的实现正是借鉴了 CronJob 的实现，特别是确定下次执行时间点的逻辑部分。
#### 核心逻辑
backupschedule 控制器的核心逻辑在 **pkg/backup/backupschedule/backup_schedule_manager.go** 文件中的 Sync 函数内。与前文提到 backup 控制器核心逻辑时一样，我们省略了不太重要的细节，只在这里呈现了主要逻辑，读者可以自行对照源码查看具体实现细节。

```func (bm *backupScheduleManager) Sync(bs *v1alpha1.BackupSchedule) error {
    defer bm.backupGC(bs)
    if err := bm.canPerformNextBackup(bs); err != nil { return err }
    scheduledTime, err := getLastScheduledTime(bs, bm.now)
    if err := bm.deleteLastBackupJob(bs); err != nil { return nil }
    backup, err := createBackup(bm.deps.BackupControl, bs, *scheduledTime)
    bs.Status.LastBackup = backup.GetName()
    bs.Status.LastBackupTime = &metav1.Time{Time: *scheduledTime}
    bs.Status.AllBackupCleanTime = nil
    return nil
}
```

这里首先调用 **canPerformNextBackup 函数**来判断是否应该创建新的 Backup 资源，来执行新的备份任务。该函数中会判断，如果上一次备份已经完成，或者上一次备份已经执行失败，则同意执行下一次备份，否则拒绝。

在决定执行备份任务后，我们会调用 **getLastScheduledTime 函数**来获取下次备份执行时间。getLastScheduledTime 的核心逻辑是，根据当前时刻 now 和 cron 格式的定时任务描述，计算出当前时刻 now 之前刚刚过去的最近一个 cron 时刻。getLastScheduledTime 的实现里面还有很多边界条件处理的逻辑，这里不再赘述，读者可以自行查看代码。

在得到备份时刻后，这里会调用 createBackup 创建 Backup 资源，从而将实际的备份任务交给 backup 控制器去完成。
## backup-manager
为了适应k8s的执行环境，我们在已有的 br、dumpling、lightning 等备份工具之上，抽象出了一个 backup-manager，来对上述工具进行**统一的入口参数封装**。上面提到的所有控制器在生成任务 Job 资源时，都会以 backup-manager 作为镜像。backup-manager 负责通过容器启动参数和对应 Backup、Restore 资源的 spec 来启动对应的工具执行备份恢复任务，同时也负责同步 Backup、Restore 资源的 status，更新目前的状态和进度。

与控制器逻辑对应，目前 backup-manager 实现了基于两套工具的备份和恢复，一种是备份和恢复都使用 br，对应 backup 和 restore 命令；一种是使用 dumpling 备份，使用 lightning 恢复，对应 import 和 export 命令。
### backup/restore
这里以 br 为例，介绍一下 backup-manager 实现备份与恢复的主要逻辑流程。
如前所述，当 backup 控制器调用 **makeBackupJob 函数**创建了备份 Job 后，Job 控制器会启动一个 Pod 来执行任务，而使用的镜像正是 backup-manager。makeBackupJob 传入的第一个容器启动参数是 backup，对应的处理逻辑在 **cmd/backup-manager/app/cmd/backup.go** 中。

启动时与控制器一样，backup-manager 会构建一系列 Informer、Lister、Updater 等常见的 Kubernetes 客户端对象，然后调用 ProcessBackup 函数，执行实际的备份操作，其主要逻辑简化如下所示。
```func (bm *Manager) ProcessBackup() error {
    backup, err := bm.backupLister.Backups(bm.Namespace).Get(bm.ResourceName)
    if backup.Spec.From == nil {
        return bm.performBackup(ctx, backup.DeepCopy(), nil)
    }
}
func (bm *Manager) performBackup(ctx context.Context, backup *v1alpha1.Backup, db *sql.DB) error {
    // update status to BackupRunning
    backupFullPath, err := util.GetStoragePath(backup)
    backupErr := bm.backupData(ctx, backup)
    // update status to BackupComplete
}
func (bo *Options) backupData(ctx context.Context, backup *v1alpha1.Backup) error {
    clusterNamespace := backup.Spec.BR.ClusterNamespace
    args := make([]string, 0)
    args = append(args, fmt.Sprintf("--pd=%s-pd.%s:2379", backup.Spec.BR.Cluster, clusterNamespace))
    dataArgs, err := constructOptions(backup)
    args = append(args, dataArgs...)
    fullArgs := []string{"backup", backupType}
    fullArgs = append(fullArgs, args...)
    klog.Infof("Running br command with args: %v", fullArgs)
    bin := path.Join(util.BRBinPath, "br")
    cmd := exec.CommandContext(ctx, bin, fullArgs...)
    // parse error messages
}
```
ProcessBackup 的主要逻辑非常简单，首先获取对应命名空间中的 Backup 对象，然后调用 performBackup。后者先获取备份的路径，目前支持 **s3、gcs 和 local** 三种路径格式，其中 local 指的是 **PV** 方式挂载的本地路径。然后这里会调用 backupData，使用 br 工具进行备份。在 backupData 中，我们首先组合出 br 工具需要的命令行参数，然后使用 “backup“ 作为 br 命令，执行 br 二进制文件，并解析可能的错误输出。当这个命令执行成功时，此次备份任务也就完成了。至此，整个备份的核心逻辑就介绍完毕了。
### import/export
这部分逻辑与上述 br 相关逻辑类似，只是换成了用 dumpling 和 lightning 来进行备份和恢复，因此不再赘述。
### clean
在 backup 控制器生成了执行 **clean** 命令的 Job 后，backup-manager 就会执行 clean 对应的逻辑，来清理对应的备份文件。这部分逻辑的入口在 **cmd/backup-manager/app/cmd/clean.go**，调用 ProcessCleanBackup 函数开始清理流程。在一系列检查之后，分别会调用 cleanBRRemoteBackupData/cleanRemoteBackupData 函数来删除存放在远端的 br/dumpling 备份文件。
## 总结
本文讲解了 TiDB Operator 提供的**备份与恢复功能的实现与设计**。当用户提交了相应的备份与恢复任务时，对应的 backup、restore 和 backupschedule 控制器会调用 backup-manager 执行实际的任务。从**抽象封装**的角度来看，backupschedule 基于 backup 封装了定时任务逻辑，而 backup-manager 则是将具体的底层工具封装成了一个统一入口，方便各控制器调用。

如果有什么好的想法，欢迎通过 [#sig-k8s](https://slack.tidb.io/invite?team=tidb-community&channel=sig-k8s&ref=pingcap-tidb-operator) 或 [pingcap/tidb-operator](https://github.com/pingcap/tidb-operator) 与 TiDB Operator 社区交流。
