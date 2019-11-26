---
title: Nginx 代理 ws/wss 请求
index_img: /img/http/nginx.png
tags: 计算机网络
date: 2019-11-25 
---

Nginx 一个强大的代理服务。Websocket 前后端持续通信的一种方案。
<!-- more -->

#### 背景

近期项目中使用 websocket 实时传递录音数据到后端进行语义分析，且后端接口需要签名参数进行鉴权。由于前端浏览器中 WebSocket 对象不能在头信息中增添鉴权信息，所以最终采用: 前端传递给 node 服务，最终由 node 发起鉴权请求到实际处理接口。

![传输结构](https://note.youdao.com/yws/api/personal/file/WEB98aab22f1d59ab4b41edbd37ff4a1837?method=getImage&version=12641&cstk=HQr78IvQ)

中间的来回传递用到了 [Nginx](http://nginx.org/) 进行接口代理。

#### 配置

```sh
server {
    listen 443 ssl; #https和wss协议默认端口
    # ssl的相关配置
    ssl on;
    ssl_certificate /xxxx/xxxx; # ssl pem文件地址
    ssl_certificate_key /xxxx/xxxx; # ssl key文件地址
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_buffer_size 1400;
    add_header Strict-Transport-Security max-age=15768000;
    add_header Cache-Control no-store;
    ssl_stapling on;
    ssl_stapling_verify on;

    server_name 域名;
    # 转发wss协议

    location / {
        proxy_pass http://127.0.0.1:3000;
        # 项目实际的地址，个人当前是一个后端SSR项目所以转走了，可以是单页面
    }

    location /wss {
        proxy_pass http://127.0.0.1:9200;  # node的 ws 地址
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;

        proxy_connect_timeout 1d;  # 超时时间 可以更改
        proxy_send_timeout 1d;
        proxy_read_timeout 1d;
    }
    # 转发https协议
    location /api {
        proxy_pass http://127.0.0.1:3000; # 以 /api 开头的其他的接口转发
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
}
```

这样的话，在访问 ```域名/wss``` 的时候就会转发到实际的 ws 地址(端口是9200)上去。访问```域名/api/xxx```会转发到端口是3000的服务上去。

#### 另一种情况

有时候，可能会有一个单独的 ws/wss 服务，不能放在一个具体的域名下进行路由的转发。则就需要单独对这个 ws/wss 服务进行转发。

在对 wss 服务转发的时候需要使用 Nginx 的 upstream 模块。

```sh
stream {
  upstream https_ws {
    server 127.0.0.1:9200; # 实际ws服务地址
  }
  server {
    listen 9300 ssl; # 对外暴露9300，当然也可以使用单独的域名，配置server_name即可
    ssl_certificate 证书地址;
    ssl_certificate_key 证书key地址;

    proxy_connect_timeout 1h;
    proxy_timeout 1h;
    proxy_pass https_ws; # 代理地址
  }
}
```
