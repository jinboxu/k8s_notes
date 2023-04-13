## 使用kubeadmin搭建高可用k8s集群

**参考文档:  https://www.cnblogs.com/ssgeek/p/11942062.html**

**这篇文档主要记录搭建k8s多主高可用的方法，完整的内容请将“k8s安装”这篇文档结合在一起阅读**



#### 1. 部署环境说明

| 主机名称      | ip地址      | 角色        |
| ------------- | ----------- | ----------- |
| -             | 172.16.0.29 | 虚拟ip(vip) |
| k8s-master-01 | 172.16.0.44 | master      |
| k8s-master-02 | 172.16.0.5  | master      |
| k8s-master-03 | 172.16.0.17 | master      |
| k8s-node-01   | 172.16.0.25 | node        |
| k8s-node-02   | 172.16.0.41 | node        |



#### 2. 集群架构

高可用主要体现在master相关组件及etcd，master中apiserver是集群的入口，搭建三个master通过keepalived提供一个vip实现高可用，并且添加haproxy来为apiserver提供反向代理的作用，这样来自haproxy的所有请求都将轮询转发到后端的master节点上。如果仅仅使用keepalived，当集群正常工作时，所有流量还是会到具有vip的那台master上，因此加上了haproxy使整个集群的master都能参与进来，集群的健壮性更强。对应架构图如下所示： 

![k8s集群架构](pics\k8s集群架构.jpg)



#### 3. 初始工作

所有节点操作:

- 所有节点修改主机名和hosts文件，文件内容如下 

```shell
172.16.0.29  master.k8s.io k8s-vip
172.16.0.44  master01.k8s.io k8s-master-01
172.16.0.5   master02.k8s.io k8s-master-02
172.16.0.17  master03.k8s.io k8s-master-03
172.16.0.25  node01.k8s.io k8s-node-01
172.16.0.41  node02.k8s.io k8s-node-02
```



- 所有节点时间同步

>  腾讯云主机已经默认配置了时间同步



- 关闭防火墙，关闭selinux,  禁用swap,  开启路由转发 ， 设置资源配置文件 
- 创建逻辑卷并将docker容器的文件目录挂载到下面(默认位置为 /var/lib/docker)

- 安装docker并设置数据目录，安装kubeadm，kubelet和kubectl

> 这些在"k8s安装"文档上都有写



修改docker的配置文件，目前k8s推荐使用的docker文件驱动是systemd

```shell
[root@k8s-master-01 ~]# cat /etc/docker/daemon.json 
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
[root@k8s-master-01 ~]# 
```



#### 4. 部署keepalived和haproxy

在三台master节点操作：

###### 4.1 keepalived部署

```shell

! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 3
    weight -30
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100                     //权重,分别是100, 90, 80
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec01
    }
    virtual_ipaddress {
        172.16.0.29 dev eth0        //虚拟ip
    }
    track_script {
        check_haproxy
    }

}
```

> k8s-master-01的配置（keepalive权重最高），其他两个主节点除了权重大小不一，其他一样



###### 4.2 haproxy部署

**配置中声明了后端代理的三个master节点服务器，指定了haproxy运行的端口为16443等，因此16443端口为集群的入口，其他的配置不做赘述。** 

```shell
[root@k8s-master-01 ~]# cat /etc/haproxy/haproxy.cfg 
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443                //16443端口为集群的入口
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      master01.k8s.io   172.16.0.44:6443 check
    server      master02.k8s.io   172.16.0.5:6443 check
    server      master03.k8s.io   172.16.0.17:6443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
[root@k8s-master-01 ~]# 
```

> 配置中声明了后端代理的三个master节点服务器，指定了haproxy运行的端口为16443等，因此16443端口为集群的入口，其他的配置不做赘述。 



###### 4.3 启动

```shell
# systemctl enable haproxy
# systemctl start haproxy
# systemctl enable keepalived
# systemctl start keepalived
```



#### 5. 安装master

在具有vip的master上操作，这里为k8s-master-01

###### 5.1 创建kubeadm配置文件

```shell
[root@k8s-master-01 ~]# mkdir /usr/local/kubernetes/manifests -p
[root@k8s-master-01 ~]# cd /usr/local/kubernetes/manifests/
[root@k8s-master-01 manifests]# vim kubeadm-config.yaml
apiServer:
  certSANs:
    - k8s-master-01
    - k8s-master-02
    - k8s-master-03
    - master.k8s.io
    - 172.16.0.44
    - 172.16.0.5
    - 172.16.0.17
    - 172.16.0.29
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "master.k8s.io:16443"         // 1
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.15.7                      //k8s版本，注意和kubeadmin一致
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16                      //pod网络
  serviceSubnet: 10.1.0.0/16                    //service网络
scheduler: {}
```



###### 5.2 初始化master节点

```shell
[root@k8s-master-01 manifests]# kubeadm init --config kubeadm-config.yaml 
```



###### 5.3 按照提示配置环境变量

```
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 其他普通用户都可这样配置来使用k8s集群



启用 kubectl 命令的自动补全功能 （略）



#### 6. master节点安装集群网络(略)



#### 7. 其他节点加入集群

##### master节点加入集群

###### 7.1 复制密钥及相关文件 

在执行init的机器，此处为k8s-master-01上有如下复制证书的脚本:

```shell
#!/bin/bash
for index in 5 17; do
  ip=172.16.0.${index}
  ssh $ip "mkdir -p /etc/kubernetes/pki/etcd; mkdir -p ~/.kube/"
  scp /etc/kubernetes/pki/ca.crt $ip:/etc/kubernetes/pki/ca.crt
  scp /etc/kubernetes/pki/ca.key $ip:/etc/kubernetes/pki/ca.key
  scp /etc/kubernetes/pki/sa.key $ip:/etc/kubernetes/pki/sa.key
  scp /etc/kubernetes/pki/sa.pub $ip:/etc/kubernetes/pki/sa.pub
  scp /etc/kubernetes/pki/front-proxy-ca.crt $ip:/etc/kubernetes/pki/front-proxy-ca.crt
  scp /etc/kubernetes/pki/front-proxy-ca.key $ip:/etc/kubernetes/pki/front-proxy-ca.key
  scp /etc/kubernetes/pki/etcd/ca.crt $ip:/etc/kubernetes/pki/etcd/ca.crt
  scp /etc/kubernetes/pki/etcd/ca.key $ip:/etc/kubernetes/pki/etcd/ca.key
  scp /etc/kubernetes/admin.conf $ip:/etc/kubernetes/admin.conf
  scp /etc/kubernetes/admin.conf $ip:~/.kube/config
done
```

> 执行此脚本将证书复制到其他两台未加进来的master节点上(先打通ssh)



###### 7.2 master加入集群 

分别在其他两台master上操作，执行在`k8s-master-01`上init后输出的join命令，如果找不到了，可以在master01上执行以下命令输出 :

```shell
[root@k8s-master-01 ~]# kubeadm token create --print-join-command
```



**执行join命令，需要带上参数`--control-plane`表示把master控制节点加入集群**



###### 7.3 检查

在其中一台master上执行命令检查集群及pod状态

```shell
[root@k8s-master-01 ~]# kubectl get node
[root@k8s-master-01 ~]# kubectl get pods --all-namespaces
```



##### node节点加入集群

分别在node节点上操作，执行`join`命令 

同理，检查



#### 8. 集群后继扩容

默认情况下加入集群的`token`是`24`小时过期，`24`小时后如果是想要新的`node`加入到集群，需要重新生成一个`token`，命令如下

```
# 显示获取token列表
$ kubeadm token list
# 生成新的token
$ kubeadm token create
```

除`token`外，`join`命令还需要一个`sha256`的值，通过以下方法计算

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

用上面输出的`token`和`sha256`的值拼接`join`命令即可(或是直接利用`kubeadm token create --print-join-command`)





#### 开启ipvs

```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
#查看是否加载
lsmod | grep ip_vs
#配置开机自加载
cat <<EOF>> /etc/rc.local
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod +x /etc/rc.d/rc.local
```



#### 补充

- ntpd服务配置的细节

```shell
server ntpupdate.tencentyun.com iburst
interface ignore wildcard
interface listen eth0
```

> 这是腾讯云服务器默认启动的ntpd服务的配置，因为ntp服务默认监听在服务器所有网卡的udp 123端口，为了避免这种情况，单独指定监听在物理网卡上



- 重启node节点的过程

```shell
## 排干准备重启node的pod
$ kubectl drain k8s-node0 --ignore-daemonsets
```



- k8s拉取harbor私有镜像

创建secret:

```shell
$ kubectl create secret docker-registry secret-name --namespace=default \
--docker-server=http://abc.qhgctech.com --docker-username=username \
--docker-password=password --dry-run -o yaml > admin.yaml
```

> 其他参考方法:  https://blog.csdn.net/qq_35959573/article/details/84859790



创建应用，拉取私有镜像:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: mysqlp-grafana
  name: mysqlp-grafana
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mysqlp-grafana
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mysqlp-grafana
    spec:
      containers:
      - image: abc.qhgctech.com/mysql_performance/grafana:v1.0
        name: grafana
      imagePullSecrets:
      - name: secret-name
```

> 在创建容器时指定 imagePullSecrets 指标，指定刚才创建的秘钥 



- 设置k8s master节点参与pod负载或不负载

**让 master节点参与POD负载:**

① 删除master节点的默认污点

```shell
$ kubectl taint nodes k8s-master node-role.kubernetes.io/master-
```

② 在pod上添加污点容忍或者增加节点亲缘性(或者使用标签选择器)

```
这时候就需要通过修改pod的配置，使其可以在任意节点上运行（包括master和node）：

    tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule


如果需要指定必须在master上执行，需要再配置nodeSelector：

    nodeSelector:
        node-role.kubernetes.io/master: ""
```



**让 master节点恢复不参与POD负载:**

```shell
$ kubectl taint nodes k8s-master node-role.kubernetes.io/master=:NoSchedule
```



**让 master节点恢复不参与POD负载，并将Node上已经存在的Pod驱逐出去:**

```shell
$ kubectl taint nodes <node-name> node-role.kubernetes.io/master=:NoExecute
```

