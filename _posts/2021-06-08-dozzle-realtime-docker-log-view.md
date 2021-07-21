---
layout: post
title: "使用dozzle通过web界面实时查看docker日志"
description: "如何通过web界面查看docker容器日志"
categories: [shell]
tags: [bash, ]
---


* Kramdown table of contents
{:toc .toc}

## 1 运行dozzle
```
docker run --detach --volume=/var/run/docker.sock:/var/run/docker.sock --net host  amir20/dozzle --addr 127.0.0.1:8080  --base /dockerlogs
```
## 2 反向代理
```
server {
    listen 80;
    server_name xxx;
    client_max_body_size 1G;
    add_header  Access-Control-Allow-Origin "https://xxx";
    add_header  Access-Control-Allow-Methods "GET, POST, OPTIONS";
    add_header  Access-Control-Allow-Headers "Origin, Authorization, Accept";
    add_header  Access-Control-Allow-Credentials true;

    location ^~ /dockerlogs {
        proxy_pass http://localhost:8080;
    }
}
```

## 3 访问

http://x.x.x.x/dockerlogs
