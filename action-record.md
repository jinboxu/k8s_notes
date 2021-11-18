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











