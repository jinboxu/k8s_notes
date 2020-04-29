## NFS动态持久卷

**github :   https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client**



#### 1. NFS Provisioner简介

NFS Provisioner 是一个自动配置卷程序，它使用现有的和已配置的 NFS 服务器来支持通过持久卷声明动态配置 Kubernetes 持久卷。

- 持久卷被配置为：namespace−{namespace}-namespace−{pvcName}-${pvName}



#### 2. External NFS驱动的工作原理

**nfs-client**

- 也就是我们这一类，它通过K8S的内置的NFS驱动挂载远端的NFS服务器到本地目录；然后将自身作为storage provider，关联storage class。当用户创建对应的PVC来申请PV时，该provider就将PVC的要求与自身的属性比较，一旦满足就在本地挂载好的NFS目录中创建PV所属的子目录，为Pod提供动态的存储服务。

**nfs**

- 与 nfs-client 不同，该驱动并不使用 k8s 的 NFS 驱动来挂载远端的 NFS 到本地再分配，而是直接将本地文件映射到容器内部，然后在容器内使用 ganesha.nfsd 来对外提供 NFS 服务；在每次创建 PV 的时候，直接在本地的 NFS 根目录中创建对应文件夹，并 export 出该子目录。利用 NFS 动态提供 Kubernetes 后端存储卷。



#### 3. 宿主机需要具备挂载的条件

**参考文档： https://jingyan.baidu.com/article/c35dbcb0bb06928917fcbc74.html**

**==> 无论从provisioner容器日志还是在宿主机上手动挂载都报如下错误：**

```
mount: wrong fs type, bad option, bad superblock on 172.16.0.31:/,
       missing codepage or helper program, or other error
       (for several filesystems (e.g. nfs, cifs) you might
       need a /sbin/mount.<type> helper program)

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
```

> yum install nfs-utils    //没有nfs文件系统



解释:

```
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.43.3
            path: /data/nfs_share
```

> pod在使用NFS卷时，是通过宿主机的文件系统挂载到pod容器目录/var/lib/kubelet/pods/{container_id}/volumes/kubernetes.io~nfs/{mountpoint}