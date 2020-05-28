## ingress  controller 注解使用

参考文档: **https://www.cnblogs.com/h-gallop/p/11984651.html **

#### Ingress annotations

官方文档手册:  **https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/**



###### 1. 为Ingress添加basic-auth认证

examples:  **https://kubernetes.github.io/ingress-nginx/examples/auth/basic/**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    kubernetes.io/name: "Prometheus"
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
spec:
  rules:
  - host: prometheus-k8s.qhgctech.com
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus
          servicePort: 9090
  - host: alertmanager-k8s.qhgctech.com
    http:
      paths:
      - path: /
        backend:
          serviceName: alertmanager
          servicePort: 80
```







###### 2. Rewirte

In some scenarios the exposed URL in the backend service differs from the specified path in the Ingress rule. Without a rewrite any request will return 404. Set the annotation `nginx.ingress.kubernetes.io/rewrite-target` to the path expected by the service.

If the Application Root is exposed in a different path and needs to be redirected, set the annotation `nginx.ingress.kubernetes.io/app-root` to redirect requests for `/`.



examples:  **https://kubernetes.github.io/ingress-nginx/examples/rewrite/**



###### 3. ingress上传文件大小限制

用ingress方式暴露出来，上传文件到应用时报"413 Request Entity Too Large"

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app-xx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 50M
spec:
...
```



#### ingress路径的问题

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: admin-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
  #  nginx.ingress.kubernetes.io/rewrite-target: /$1
  #  kubernetes.io/ingress.class: "nginx"
  #  nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - host: portal.xxxx.com
    http:
      paths:
      #- path: /admin/(.+)
      #- path: /admin/($/.*)
      - path: /admin(/|$)(.*)
        backend:
          serviceName: app-server
          servicePort: 80
```

> 很明显，/admin(/|$)(.*)匹配的最准确





