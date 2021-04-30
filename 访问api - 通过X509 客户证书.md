## 访问api - 通过X509 客户证书

这篇文档记录了：**通过X509 客户证书进行身份验证，使用RBAC进行权限控制，同时通过kubeconfig(定义集群、用户和上下文)进行管理**

#### 1. 用户认证

https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/



*X509 客户证书*

通过给 API 服务器传递 `--client-ca-file=SOMEFILE` 选项，就可以启动客户端证书身份认证。 所引用的文件必须包含一个或者多个证书机构，用来验证向 API 服务器提供的客户端证书。 如果提供了客户端证书并且证书被验证通过，则 subject 中的公共名称（Common Name）就被 作为请求的用户名。 自 Kubernetes 1.4 开始，客户端证书还可以通过证书的 organization 字段标明用户的组成员信息。 要包含用户的多个组成员信息，可以在证书种包含多个 organization 字段。  -- 引用官方

例如，使用 `openssl` 命令行工具生成一个证书签名请求：

```bash
openssl req -new -key jbeda.pem -out jbeda-csr.pem -subj "/CN=jbeda/O=app1/O=app2"
```

此命令将使用用户名 `jbeda` 生成一个证书签名请求（CSR），且该用户属于 "app" 和 "app2" 两个用户组。



#### 2. 创建一个用户，对集群命名空间foo有查看服务的权限

https://blog.csdn.net/wangmiaoyan/article/details/102549310

```shell
#### 1. 生产私钥
$ openssl genrsa -out viewAll.key 2048
$ openssl req -new -key viewAll.key -out viewAll.csr -subj "/CN=viewAll"


#### 2. CA去签署
$ openssl x509 -req -in viewAll.csr -CA ./ca.crt -CAkey ./ca.key -CAcreateserial -out viewAll.crt -days 365
Signature ok
subject=/CN=viewAll
Getting CA Private Key

$ openssl x509 -in viewAll.crt -text -noout   //查看
```



```shell
#### 3. RBAC授权，下面定义了一个role，它允许用户获取并列出foo命名空间中的服务
[k8s@k8s-master RBAC]$ cat role.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["pods", "services"]
$ kubectl -n foo create rolebinding test --role=service-reader --user=viewAll
```

> 注意，这里没有对集群用于view权限进行授权，下面会通过另一种方式进行介绍





```shell
#### 4. kubeconfig配置
https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/

## 4.1 创建名为config.yaml的文件
$ cat config.yaml 
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: development

users:
- name: viewAll

contexts:
- context:
  name: view-all

## 4.2 定义集群、用户和上下文
将集群信息添加到配置文件中：
​```
$ kubectl config --kubeconfig=config.yaml set-cluster development --server=https://192.168.0.232:6443 --certificate-authority=../ca.crt --embed-certs=true
​```
> --embed-certs=true 表示将数据存放在文件中，默认是文件名


将用户的信息添加到配置文件中：
​```
$ kubectl config --kubeconfig=config.yaml set-credentials viewAll --client-certificate=../viewAll.crt --client-key=../viewAll.key --embed-certs=true
​```


将上下文信息添加到配置文件中:
​```
$ kubectl config --kubeconfig=config.yaml set-context view-all --cluster=development --namespace=foo --user=viewAll
​```

设置更改当前的上下文:
​```
$ $ kubectl config --kubeconfig=config.yaml use-context view-all
​```
[k8s@k8s-master kubeconfigDir]$
```

![访问api](pics\访问api.jpg)





#### 3. 证书签名请求

https://kubernetes.io/zh/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user



上面的方法，自己手动通过CA去签署，但是如果我们在一个master不受我们管理的集群中，比如 腾讯云的TKE托管的容器服务，我们是拿不到CA的私钥的，这时我们就可以通过**证书签名请求**的方式对我们的请求文件进行签署。



步骤与上面手动创建的类似:

##### 3.1 创建私钥，略

##### 3.2 创建 CertificateSigningRequest

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: viewAll
spec:
  groups:
  - system:authenticated
  request: xxxxxx                            // CSR文件内容的base64编码值
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

- `usage` 字段必须是 '`client auth`'
- `request` 字段是 CSR 文件内容的 base64 编码值。 要得到该值，可以执行命令 `cat john.csr | base64 | tr -d "\n"`



批准证书签名请求：

*类似于手动通过CA签署*

```shell
$ kubectl get csr
$ kubectl certificate approve viewAll
## 取得证书
$ kubectl get csr/viewAll -o yaml
```

> 证书的内容使用 base64 编码，存放在字段 `status.certificate`



##### 3.3 创建viewAll用户，拥有集群的view权限

使用k8s集群自带的clusterrole  view角色

```shell
$ kubectl create clusterrolebinding view-all --clusterrole=view --user=viewAll
```



#### 3.4 添加到kubeconfig

添加凭据:

```shell
$ kubectl config --kubeconfig=config.yaml set-credentials viewAll --client-key=../viewAll.key --embed-certs=true
```

**证书的内容使用base64编码，我们在上面3.2步骤批准证书时，已经自动生成了证书的base64 编码，所以这里我们在添加凭据的时候不需要通过--client-certificate指定签名文件(避免了解码又转码的过程)，只需要在文件中手动添加client-certificate-data字段内容即可。**





