---
title: "Nginx的安装和简单配置"
date: 2020-09-05T22:11:09+08:00
lastmod: 2020-09-06T01:52:46+08:00
draft: false
keywords: ['Nginx','Nginx配置', '安装Nginx','Linux安装nginx']
description: "通过官方仓库安装nginx的最新版本，并做简单配置，并提供静态网页"
tags: ['nginx']
categories: ['网页服务器']
author: ""
---

通过官方仓库来安装nginx的最新版本，做简单配置，并提供静态网页
<!--more-->

## 安装nginx

采用官方仓库安装比较简单快速，没必要在纠结选择那种，如果是内网环境，无法上网，可以通过一台可以上网的服务器，下载rpm包上传到提供服务的服务器上来安装。

### 添加官方仓库

```bash
sudo su
cat > /etc/yum.repos.d/nginx.repo<<EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
```

### 安装nginx

```bash
yum install nginx -y
systemctl enable nginx
systemctl start nginx
```

## 更改配置

nginx的配置存放在/etc/nginx目录下边, nginx.conf 是全局的一些配置，config.d目录是网站配置，一般建议一个域名对应一个文件

1. 备份/etc/nginx/conf.d/default.conf

```bash
sudo cp /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
```

2. 修改/etc/nginx/conf.d/default.conf

```bash
cat > /etc/nginx/conf.d/default.conf<<EOF
# 使用http协议访问带www的网址，跳转到https协议的带www的网址上
server {
    listen 80;
    server_name www.net-cc.com;
    return 301 https://www.net-cc.com$request_uri;
}

# 使用http协议访问不带www的网址，跳转到https协议的带www的网址上
server {
    listen 80;
    server_name net-cc.com;
    return 301 https://www.net-cc.com$request_uri;
}

# 使用https协议访问不带www的网址，跳转到https协议的带www的网址上
server {
    listen 443 ssl;
    server_name net-cc.com;
    return 301 https://www.net-cc.com$request_uri;

    # SSL证书配置
    ssl_certificate /etc/nginx/ssl/net-cc.com.pem;  # 证书
    ssl_certificate_key /etc/nginx/ssl/net-cc.com.key; # 证书的密钥
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  # 协议
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; # 加密方式
    ssl_session_timeout  5m; 
}

# https协议的带www的网址（主）配置
server {
    listen       443 ssl;
    server_name  www.net-cc.com;  # 网站的域名

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    # SSL证书配置
    ssl_certificate /etc/nginx/ssl/www.net-cc.com.pem;  # 证书
    ssl_certificate_key /etc/nginx/ssl/www.net-cc.com.key; # 证书的密钥
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  # 协议
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; # 加密方式
    ssl_session_timeout  5m; 

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    # 拒绝访问.git目录
    location ~ /\.git {
        deny  all;
    }
}
EOF
```

## 申请https证书

我是通过阿里云申请的证书，可以使用Let's Encrypt申请免费三个月的证书

## 下载网站的静态文件

```
git clone git@gitlab.com:net-cc/net-cc.com.git
```
