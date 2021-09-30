---
title: sl-im 基于 Swoft 微服务协程框架和 Layim 网页聊天系统 开发出来的聊天室
tags: []
id: '72'
categories:
  - - uncategorized
date: 2020-09-23 07:51:02
---

![](http://qiniu.gaobinzhan.com/2020/04/13/562596a1c87ac.png?imageView2/2/w/300)

![](http://img.shields.io/badge/php-%3E=7.1-brightgreen.svg?maxAge=2592000)![](http://img.shields.io/badge/swoole-%3E=4.3.3-brightgreen.svg?maxAge=2592000)![](http://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)

## 简介

sl-im 是基于 Swoft 微服务协程框架和 Layim 网页聊天系统 所开发出来的聊天室。

当前分支为2.x开发版本，如需部署，请下载releases

## 体验地址

sl-im https://im.gaobinzhan.com

## 演示图

![](http://qiniu.gaobinzhan.com/2020/04/13/a96b031c660ca.jpg)

## 功能

*   登录注册（Http）
*   单点登录（Websocket）
*   私聊（Websocket）
*   群聊（Websocket）
*   在线人数（Websocket）
*   获取未读消息（Websocket）
*   好友在线状态（Websocket）
*   好友 查找 添加 同意 拒绝（Http+Websocket）
*   群 创建 查找 添加 同意 拒绝（Http+Websocket）
*   聊天记录存储
*   心跳检测
*   消息重发
*   断线重连

## Requirement

*   PHP 7.1+
*   Swoole 4.3.4+
*   Composer
*   Swoft >= 2.0.8

## 部署方式

### Composer

```
composer update
```

### bean

`app/bean.php`

```
'db' => [
        'class'    => Database::class,
        'dsn'      => 'mysql:dbname=im;host=127.0.0.1:3306',
        'username' => 'root',
        'password' => 'gaobinzhan',
        'charset'  => 'utf8mb4',
    ],
'db.pool' => [
        'class'     => \Swoft\Db\Pool::class,
        'database'  => bean('db'),
        'minActive' => 5, // 自己调下连接池大小
        'maxActive' => 10
    ],
```

### 数据表迁移

`php bin/swoft mig:up`

### env配置

`vim .env`

```
# basic
APP_DEBUG=0
SWOFT_DEBUG=0

# more ...
APP_HOST=https://im.gaobinzhan.com/
WS_URL=ws://im.gaobinzhan.com/im
# 是否开启静态处理 这里我关了 让nginx去处理
ENABLE_STATIC_HANDLER=false 
# swoole v4.4.0以下版本, 此处必须为绝对路径
DOCUMENT_ROOT=/data/wwwroot/IM/public
```

### nginx配置

```
server{
    listen 80;
    server_name im.gaobinzhan.com;
    return 301 https://$server_name$request_uri;
}

server{
    listen 443 ssl;
    root /data/wwwroot/IM/public/;
    add_header Strict-Transport-Security "max-age=31536000";
    server_name im.gaobinzhan.com;
    access_log /data/wwwlog/im-gaobinzhan-com.access.log;
    error_log /data/wwwlog/im-gaobinzhan-com.error.log;
    client_max_body_size 100m;
    ssl_certificate /etc/nginx/ssl/full_chain.pem;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    location / {
        proxy_pass http://127.0.0.1:9091;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    location /im {
        proxy_pass http://127.0.0.1:9091;
        proxy_http_version 1.1;
        proxy_read_timeout   3600s;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    location ~ .*\.(jsicocssttfwoffwoff2pngjpgjpegsvggifhtm)$ {
        root /data/wwwroot/IM/public;
    }
}
```

### Start

```
php bin/swoft ws:start
```

```
php bin/swoft ws:start -d
```

怎么访问还用写吗？？？点个star吧 ✌️

## 联系方式

*   WeChat：gaobinzhan
*   QQ：975975398

我的博客即将同步至腾讯云+社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite\_code=14q93ezyewy0r