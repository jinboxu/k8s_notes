## nginx configuration

#### 1. 4层代理

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

