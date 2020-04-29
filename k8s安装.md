## k8s安装

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具 :

```shell
# 创建一个 Master 节点
$ kubeadm init
# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```



**参考文档:  https://www.cnblogs.com/itzhao/p/11377216.html**

​                  **https://www.cnblogs.com/ssgeek/p/11942062.html**

#### 初始工作

```shell
关闭防火墙：
# systemctl stop firewalld
# systemctl disable firewalld

关闭selinux：
# sed -i 's/enforcing/disabled/' /etc/selinux/config
# setenforce 0

关闭swap：
# swapoff -a $ 临时
# vim /etc/fstab $ 永久

添加主机名与IP对应关系（记得设置主机名）：
# cat /etc/hosts
192.168.0.11 k8s-master
192.168.0.12 k8s-node1
192.168.0.13 k8s-node2

将桥接的IPv4流量传递到iptables的链：
# cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# modprobe br_netfilter
# sysctl --system    //sysctl -p /etc/sysctl.d/k8s.conf

修改系统限制
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```





#### 安装kubeadm，kubelet和kubectl 

**安装版本： 1.15.7**

添加阿里云YUM源

```shell
# cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF
# yum install -y kubelet-1.15.7 kubeadm-1.15.7 kubectl-1.15.7
# systemctl enable kubelet
```

> kubelet 运行在 Cluster 所有节点上，负责启动 Pod 和容器。
>
> kubeadm 用于初始化 Cluster。
>
> kubectl 是 Kubernetes 命令行工具。通过 kubectl 可以部署和管理应用，查看各种资源，创建、删除和更新各种组件。



#### 部署K8S  Master

```shell
# kubeadm init \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.15.7 \
--apiserver-advertise-address=192.168.0.232 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```

> image-repository string：这个用于指定从什么位置来拉取镜像（1.13版本才有的），默认值是k8s.gcr.io，我们将其指定为阿里云镜像地址 
>
> –apiserver-advertise-address 指明用 Master 的哪个 interface 与 Cluster 的其他节点通信 
>
> –pod-network-cidr指定 Pod 网络的范围。Kubernetes 支持多种网络方案，而且不同网络方案对  –pod-network-cidr有自己的要求，这里设置为10.244.0.0/16 是因为我们将使用 flannel 网络方案 



**执行kubeadm init集群初始化时遇到：** 

[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". 

[警告IsDockerSystemdCheck]：检测到“cgroupfs”作为Docker cgroup驱动程序。 推荐的驱动程序是“systemd” 



解决方法：更换驱动

> 在/etc/docker下创建daemon.json并编辑：
>
> vim /etc/docker/daemon.json
> 加入以下内容：
>
> {
> "exec-opts":["native.cgroupdriver=systemd"]
> }



**使用kubectl工具:**

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



启用 kubectl 命令的自动补全功能 ：

```shell
 echo "source <(kubectl completion bash)" >> ~/.bashrc
```

> _get_comp_words_by_ref: command not found 报错:  
>
> ```shell
> # yum install bash-completion -y   
> 
> $ source /usr/share/bash-completion/bash_completion  
> ```



**安装pod网络插件(CNI)**

```shell
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```



#### 加入K8S Node

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令： 

```shell
# kubeadm join 192.168.0.232:6443 --token 8nye2l.vv4jscc3je7r5k0z \
--discovery-token-ca-cert-hash         sha256:0255588a32ce54e28d04d4033bdeb0ac175507154c36ff0b3019e31398783367
```



#### 查看nodes与测试kubernetes集群 

```
$ kubectl get pod -n kube-system -o wide
```

> 这里需要等一会，node节点才会变成Ready状态，因为node节点需要下载三个镜像flannel  kube-proxy pause 



在Kubernetes集群中创建一个pod，验证是否正常运行： 

```shell
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

