## nginx实现的一些细节

#### 1. 服务网关即路由功能，'/'的意义

nginx反向代理proxy_pass url后加‘/’与不加'/'的区别：

**在nginx中配置proxy_pass反向代理时，当在后面的url加上了/，nginx不会把location中匹配的路径部分代理走;如果没有/，则会把匹配的路径部分也给代理走。**

> 例如：访问的路径为/pss/bill.html
>
> - 当nginx配置文件proxy_pass后边的url带"/"时，代理到后端的路径为：http://127.0.0.1:18081/bill.html，省略了匹配到的/pss/路径；
>
> ```
> location /pss/ {
>     proxy_pass http://127.0.0.1:18081/;
> }
> ```
>
> - 当nginx配置文件proxy_pass后边的url不带"/"时，代理到后端的路径为：http://127.0.0.1:18081/pss/bill.html，连同匹配到的/pss/路径，一起进行反向代理；
>
> ```
> location /pss/ {
>     proxy_pass http://127.0.0.1:18081;
> }
> ```



**实例：**

```shell
upstream gateway-server {
    server 10.170.40.53:80;
    server 10.170.40.55:80;
    keepalive 16;
}

upstream eureka-server {
    server 10.170.40.53:8100;
    server 10.170.40.55:8100;
    server 10.170.40.56:8100;
    keepalive 16;
}

server {
    listen 80;
    server_name 10.170.40.56;
    
    location /admin {
        alias /usr/share/nginx/html;
        #index index.html index.htm;
        #rewrite ^/admin $root permanent;
    }

    location ~ ^/api-.* {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://gateway-server;        //2
    } 

    location  /eureka-server {
        proxy_pass http://eureka-server/;        //1
    }
    
}
```

> 1. 路由到eureka服务上：
>
> 需要代理的后端路径为/，所以此处proxy_pass url必须带“/” 。最终的请求路径为: http://eureka-server/
>
> 2. 路由到gateway上：
>
> 需要代理的后端路径为/api-.* ,  所以此处proxy_pass url必须不带“/” 。最终的请求路径为: http://gateway-server/api-{something} 



nginx ingress如何暴露上面的服务呢？

**ingress默认会将请求的路径传到到后端服务路径，即proxy_pass url默认没有带"/"。**所以：

- 对于eureka服务，ingress通过rewrite重写到/

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: eureka-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /       #重写到/
spec:
  tls:
  - hosts: 
    - www.test.com
    secretName: tls-secret
  rules:
  - host: www.test.com 
    http: 
      paths:
      - path: /eureka-server
        backend:
          serviceName: eureka-server
          servicePort: 8100
```

- 对于gateway服务，ingress无需做额外处理

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gateway-server-ingress
  annotations: 
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'   
    ingress.kubernetes.io/location-snippet: |
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
spec:
  tls:
  - hosts: 
    - www.test.com
    secretName: tls-secret
  rules:
  - host: www.test.com
    http:
      paths:
      - path: /api-.*
        backend:
          serviceName: gateway-server
          servicePort: 80
```



进一步思考对比ingress与传统nginx暴露服务的区别和相互转化：

![暴露服务](pics\暴露服务.jpg)

> 从上图可以看到除nginx服务之外，其他服务的暴露过程基本一致(从上面的eureka服务和gateway服务可以看出)。在传统服务下面,nginx作为静态文件的代理是直接可访问的，k8s集群下是通过ingress暴露出去的。



实例：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: admin-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /$1      ##重写
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts: 
    - www.test.com
    secretName: tls-secret
  rules:
  - host: www.test.com 
    http: 
      paths:
      - path: /admin/(.+)
        backend:
          serviceName: admin-server
          servicePort: 80
```

```shell
    location /admin {
        alias /usr/share/nginx/html;
        #index index.html index.htm;
    }
```

> 如上，用户访问/admin作为页面门户的入口



#### 2. 获取用户真实的ip

**X-Forwarded-For和相关几个头部的理解**

- **$remote_addr**
   是nginx与客户端进行TCP连接过程中，获得的客户端真实地址. Remote Address 无法伪造，因为建立 TCP 连接需要三次握手，如果伪造了源 IP，无法建立 TCP 连接，更不会有后面的 HTTP 请求
- **X-Real-IP**
   是一个自定义头。X-Real-Ip 通常被 HTTP 代理用来表示与它产生 TCP 连接的设备 IP，这个设备可能是其他代理，也可能是真正的请求端。需要注意的是，X-Real-Ip 目前并不属于任何标准，代理和 Web 应用之间可以约定用任何自定义头来传递这个信息
- **X-Forwarded-For**
   X-Forwarded-For 是一个扩展头。HTTP/1.1（RFC 2616）协议并没有对它的定义，它最开始是由 Squid 这个缓存代理软件引入，用来表示 HTTP 请求端真实 IP，现在已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用，并被写入 RFC 7239（Forwarded HTTP Extension）标准之中.



X-Forwarded-For请求头格式非常简单: ```X-Forwarded-For:client, proxy1, proxy2```

如果一个 HTTP 请求到达服务器之前，经过了三个代理 Proxy1、Proxy2、Proxy3，IP 分别为 IP1、IP2、IP3，用户真实 IP 为 IP0，那么按照 XFF 标准，服务端最终会收到以下信息:

```
X-Forwarded-For: IP0, IP1, IP2
```

详细分析一下，这样的结果是经过这样的流程而形成的：

1. 用户IP0---> 代理Proxy1（IP1），Proxy1记录用户IP0，并将请求转发个Proxy2时，带上一个Http Header
    `X-Forwarded-For: IP0`
2. Proxy2收到请求后读取到请求有 `X-Forwarded-For: IP0`，然后proxy2 继续把链接上来的proxy1 ip**追加**到 X-Forwarded-For 上面，构造出`X-Forwarded-For: IP0, IP1`，继续转发请求给Proxy 3
3. 同理，Proxy3 按照第二部构造出 `X-Forwarded-For: IP0, IP1, IP2`,转发给真正的服务器，比如NGINX，nginx收到了http请求，里面就是 `X-Forwarded-For: IP0, IP1, IP2` 这样的结果。所以Proxy 3 的IP3，不会出现在这里。
4. nginx 获取proxy3的IP 能通过![remote_address获取到，因为这个](https://math.jianshu.com/math?formula=remote_address%E8%8E%B7%E5%8F%96%E5%88%B0%EF%BC%8C%E5%9B%A0%E4%B8%BA%E8%BF%99%E4%B8%AA)remote_address就是真正建立TCP链接的IP，这个不能伪造，是直接产生链接的IP。**$remote_address  无法伪造，因为建立 TCP 连接需要三次握手，如果伪造了源 IP，无法建立 TCP 连接，更不会有后面的 HTTP 请求。**



```nginx
server { 
        listen       80; 
        server_name  localhost;
 
        #charset koi8-r; 
        #access_log  logs/host.access.log  main;
 
        location / { 
            root   html;
            index  index.html index.htm;
                            proxy_pass                  http://backend; 
 
           proxy_redirect              off;
 
           proxy_set_header            Host $host; 
           proxy_set_header            X-real-ip $remote_addr;
           proxy_set_header            X-Forwarded-For $proxy_add_x_forwarded_for;
 
        }
```

> 引用X-Forwarded-For时要这样$http_x_forwarded_for



**详细分析参考文档:  https://blog.csdn.net/xqhys/article/details/81782633**



#### 3. return和rewrite

参考文档:     https://blog.csdn.net/u010982507/article/details/104025717

###### return指令

- 停止处理请求，直接返回响应码或重定向到其他URL
- 执行return指令后，location中后续指令将不会被执行

语法:

```
return code [text];
return code URL; 
return URL;
```



###### rewrite指令

rewrite flag标志:

- last, 完成重写指令的处理，然后搜索相应的URI和位置
- break, 完成重写指令的处理，停止处理后续rewrite指令集；当前location内剩余非rewrite语句和location外的非rewrite语句可以执行。
- redirect, 返回302临时重定向，地址栏会显示跳转后的地址
- permanent, 返回301永久重定向，地址栏会显示跳转后的地址；即表示如果客户端不清除浏览器缓存，那么返回的结果将一直保存在客户端浏览器中。



```nginx
server {
      listen       80;
      server_name  localhost;
      charset      utf-8;
      location /images {
          rewrite /images/(.*) /download/$1 last;
          return 200;
      }

     location /download/ {
         root /home/nginx;
         rewrite /download/(.*) /pics/$1;
         return 200;
     }
     location /pics/ {
       return 200;
    }
}
```

> - 第一个`/images`的rewrite指令中有last指令，所以会继续当前server中的location重写匹配，会直接跳到`/download/`中，而第一个return则不会执行；
> - 在`/download/`中，rewrite指令后面没有last执行，则会顺序执行到return 200；语句，而不会继续到/pics/下；
> - 如果在`/download/`的rewrite中加入break语句，则会显示pics中的内容，而不会显示 /pics/中的return 200。如：
>
> ```nginx
> location /download/ {
>     root /home/nginx;
>     rewrite /download/(.*) /pics/$1 break;
>     return 200;
> }
> location /pics/ {
>   return 200;
> }
> ```



#### 4. Nginx下 return 和 rewrite在 301重定向上的区别

return code URL;中的code一般是重定向状态码。

重定向状态码有：

- 301：永久重定向（HTTP1.0标准）
- 302：临时重定向，禁止被缓存（HTTP1.0标准）
- 303：临时重定向，禁止被缓存，允许改变方法（HTTP2.0标准）
- 307：临时重定向，禁止被缓存，不允许改变方法（HTTP2.0标准）
- 308：永久重定向，不允许改变方法（HTTP2.0标准）



从上文rewrite可以实现301永久重定向，它与return做重定向时的区别在哪里呢?

https://www.liaosam.com/difference-of-return-and-rewrite-nginx-301-redirect.html



唯一的区别：正则匹配的性能区别：

1. rewrite ^/(.*)$ https://example.com/$1;
2. rewrite ^ https://example.com$request_uri? permanent;
3. **return 301 https://example.com$request_uri;**

第一种 rewrite 写法是抓取所有的 URI 再减去开头第一个 / (反斜线)。

第二种写法用了$request_uri 省去了减去开头第一个反斜线的过程，正则匹配上性能更优。但仍不如第三种写法，因为 rewrite 有很多写法和规则，执行到最后 nginx 才知道这是一个 301 永久重定向。

第三种则直接 return 301 + $request_uri，直接告诉 nginx 这是个 301重定向，直接抓取指定URI。

所以以上三种写法，第三种性能更优一些。当然，一般情况下快那么一点点，作为普通业余站长来说，我们也感觉不到。



#### 5.  4层代理

https://www.cnblogs.com/xiaopaipai/p/10070668.html



**通过nginx对gitlab做http代理和ssh协议代理**

```nginx
...

stream {
    # include stream.conf;
  
    upstream gitlab {
        server 10.170.40.56:222;
    }
    
    server {
        listen 222;
        proxy_connect_timeout 5;
        proxy_pass gitlab;
    }
}

http {
    ...
}
```

> stream与http块在同一级



conf.d/gitlab.conf :

```nginx
upstream gitlab_server {
    server 10.170.40.56:8008;
    keepalive 16
}

server {
    listen 443 ssl;
    server_name gitlab-inside.xxx.com
    
    # 多个二级域名使用统一证书
    include conf.d/ssl_certificate.conf
    
    location / {
        proxy_pass http://gitlab_server;
    }
    
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}

server {
    listen 80;
    server_name gitlab-inside.xxx.com
    rewrite ^/(.*)$ https://$server_name/$1 permanent;
}
```





#### 6. proxy_set_header设置Host为$proxy_host，$host与$local_host的区别

https://www.cnblogs.com/goloving/p/13663843.html

```nginx
// 如果想让Host是crmtest.aty.sohuno.com，则进行如下设置：
proxy_set_header Host crmtest.aty.sohuno.com;

// 如果不想改变请求头“Host”的值，可以这样来设置：
proxy_set_header Host $http_host;

// 但是，如果客户端请求头中没有携带这个头部，那么传递到后端服务器的请求也不含这个头部。 
// 这种情况下，更好的方式是使用$host变量——它的值在请求包含“Host”请求头时为“Host”字段的值，在请求未携带“Host”请求头时为虚拟主机的主域名：
proxy_set_header Host       $host;

// 此外，服务器名可以和后端服务器的端口一起传送：
proxy_set_header Host $host:$proxy_port;

// 如果某个请求头的值为空，那么这个请求头将不会传送给后端服务器：
proxy_set_header Accept-Encoding "";
```





JumpServer反向代理：https://docs.jumpserver.org/zh/master/admin-guide/proxy/#2-nginx

在一开始没有配置```proxy_set_header Host       $host;```之前jumpserver无法被正常访问 =>

查看jumpserver的nginx配置:

![server_name](..\pics\server_name.png)

> 原来没有传递Host头无法被匹配
>
> server_name _       表示匹配所有域名，
> server_name “”      会匹配没有传递Host头部的情况。



