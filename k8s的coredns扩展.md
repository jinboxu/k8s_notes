## k8s的coredns扩展

https://coredns.io/plugins/

coredns已经成为默认dns了。之前是kube-dns。coredns是一个灵活，可扩展的DNS服务器，可以作为Kubernetes集群DNS。与Kubernetes一样，CoreDNS项目由CNCF主持。但是在实际使用中，需要一些注意的地方。



--- 1,2,3 参考: https://segmentfault.com/a/1190000020403096 ---

#### 1. 增加应用的反亲和性，防止coredns调度到一台主机上



#### 2. 如何利用coredns 禁用ipv6的解析

如果K8S集群宿主机没有关闭IPV6内核模块的话，容器请求coredns时的默认行为是同时发起IPV4和IPV6解析。

```
template ANY AAAA {
    rcode NXDOMAIN
}
```



#### 3. 配置sub domain和upstream nameserver

在实际场景中，我们经常会有自己的内部dns服务器，例如我们的Consul域服务器位于10.150.0.1，并且所有Consul名称都具有后缀.consul.local。要在CoreDNS中配置它，集群管理员在CoreDNS ConfigMap中创建以下配置：

```
consul.local:53 {
        errors
        cache 30
        forward . 10.150.0.1
    }
```



通过使用腾讯云的Private dns，实现内外网解析:

```
qhgctech.com:53 {
    errors
    cache 30
    forward . 183.60.83.19 183.60.82.98
}
```

> 在 Private dns 环境下，那会优先走 Private dns，但是这些私有域名公网是无法访问的，只有这个vpc下的资源才可以访问，公网的话，会走DNSPod

---



