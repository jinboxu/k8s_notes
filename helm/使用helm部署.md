## 使用helm部署

```
## 下载和安装
// https://github.com/helm/helm/releases/tag/v3.4.2

## 自动补全
source <(kubectl completion bash)
source <(helm completion bash)
```



#### 1. 高可用的harbor仓库

```
$ helm repo add stable https://charts.helm.sh/stable
$ helm search repo stable/redis-ha  -l --max-col-width=200
$ helm pull stable/redis-ha --version=3.0.5

$ helm install redis-ha . -n educationcloud-pro -f values.yaml
```

