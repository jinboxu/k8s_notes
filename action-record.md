## action record

#### pod驱逐

在使用k8s集群过程中，可能会遇到节点异常或需要节点升级的情况，但又不能影响节点中服务的正常运行，就要涉及到对pod信息迁移和node节点维护。



```shell
$ kubectl cordon <k8s-node>    # 设置节点不可调度
$ kubectl drain <k8s-node> --delete-local-data --ignore-daemonsets
---
$ kubectl uncodon <k8s-node>   # 恢复使节点可调度
```

> --delete-local-data:  即使pod使用了emptyDir也删除
>
> --ignore-daemonsets: 忽略deamonset控制器的pod
>
> --force:  不加force参数只会删除该NODE上由ReplicationController, ReplicaSet, DaemonSet,StatefulSet or Job创建的Pod，加了后还会删除’裸奔的pod’(没有绑定到任何replication controller)



干扰:  https://kubernetes.io/zh/docs/concepts/workloads/pods/disruptions/





#### job  and cronjob

##### kubernetes CronJob 工作原理

![cronjob](pics\cronjob.png)

1. 用户创建 CronJob 类型的资源
2. CronJobController 每隔 10s 串行遍历所有的 CronJob 资源，判断是否有需要调度的 CronJob，如果有且条件允许，创建对应的 Job 资源
3. JobController 监听到有 Job 被创建，则根据 Job 创建对应的 Pod 资源
4. kube-scheduler 负责将 Pod 调度到可运行的节点上
5. Kubelet 负责在节点上创建 Pod 运行环境，启动容器



##### CronJob 资源配置字段

- concurrencyPolicy：当上一个调度时间点创建的 Job 还在执行时，当前调度时间点的调度策略，CronJobController 提供了三种类型的策略
  - Allow：在调度时间点时始终进行调度（默认值）
  - Forbid：不调度
  - Replace：删除对应的所有活跃的 Job，再创建新的 Job
- failedJobsHistoryLimit：保留最近多少个失败的 Job，默认值 1
- successfulJobsHistoryLimit：保留最近多少个成功的 Job，默认值 3
- schedule：调度时间格式，参考 https://en.wikipedia.org/wiki/Cron
- startingDeadlineSeconds：影响到 CronJob 在其调度时间点上能否进行调度，见下文详解
- suspend：是否挂起
  - true：不再调度新的 Job
  - false：正常执行调度（默认值）
- jobTemplate：Job 模板，下文讲解



参考文档:   http://yangxikun.github.io/kubernetes/2020/09/29/kubernetes-cronjob.html



手动触发cronjob:

```
kubectl create job <job-name> --from=cronjob/<cronjob-name>    #需要验证
```







