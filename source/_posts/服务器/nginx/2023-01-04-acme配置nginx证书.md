---
title: acme配置nginx证书
date: 2023-01-04 12:14:02
categories:
  - 服务器
  - nginx
tags:
  - nginx
---

## 1、命令配置

```
# 下载
curl https://get.acme.sh | sh
# 别名
alias acme.sh=~/.acme.sh/acme.sh
```

## 2、申请证书

```
# 指定dns
acme.sh --issue --server letsencrypt --dns dns_dp -d xichangyou.com -d www.xichangyou.com --webroot /home/leellun/webroot
# 指定webroot
acme.sh --issue --server letsencrypt  -d xichangyou.com -d www.xichangyou.com --webroot /home/leellun/webroot
```

## 3、安装证书

```
acme.sh --install-cert -d xichangyou.com -d www.xichangyou.com --key-file /usr/local/nginx/certs/xichangyou.com.key --fullchain-file /usr/local/nginx/certs/xichangyou.com.pem
```

## 4、配置nginx

在/usr/local/nginx/conf.d目录下 ，创建xichangyou.com.conf配置文件

```
    server {
        listen       443 ssl;
        server_name  localhost;

        ssl_certificate      /usr/local/nginx/certs/xichangyou.com.pem;
        ssl_certificate_key  /usr/local/nginx/certs/xichangyou.com.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
		location ^~ /.well-known/acme-challenge/ {
    		default_type "text/plain";
    		root /usr/local/nginx/www/html;
		}

        location / {
            root   /usr/local/nginx/html;
            index  index.html index.htm;
        }
    }
```

在nginx配置文件目录conf中引入

nginx.conf内容：

```
http{
    ....
    include /usr/local/nginx/conf.d/*.conf;
}
```

## 5 设置自动更新

```
acme.sh --upgrade --auto-upgrade
```

