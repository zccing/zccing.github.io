---
title: "安装最新版的nodejs"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['mysql', 'mysql安装','二进制安装mysql','编译安装mysql']
description: "linux安装最新版的mysql记录，通过存储库、二进制、编译三种方式进行安装mysql"
tags: ['linux']
categories: ['linux']
---

通过存储库、二进制、编译三种方式进行安装nodejs的安装笔记
<!--more-->


# nodejs安装笔记

## 安装nodejs

### 二进制安装

* 下载安装

    ```sh
    # 下载
    wget https://nodejs.org/dist/v10.15.1/node-v10.15.1-linux-x64.tar.xz
    # 解压缩
    tar -xf node-v10.15.1-linux-x64.tar.xz -C /opt/
    # 更改文件名称
    cd /opt/
    mv node-v10.15.1-linux-x64/ nodejs
    ```

* 修改环境变量

    ```sh
    cat >> /etc/profile <<EOF
    export NODEJS_HOME=/opt/nodejs/bin
    export PATH=\$NODEJS_HOME:\$PATH
    EOF
    ```

* 添加sudo权限

    ```sh
    sudo ln -s /opt/nodejs/bin/* /usr/bin/
    ```

* 更改selinux权限

* 验证

    ```sh
    node -v
    npm version
    npx -v
    ```

### 编译安装（时间关系，没有编译过）

> 待补充

### docker安装

> 待补充

## 安装配置PM2

* 安装

    ```sh
    sudo npm install -g pm2
    ```

* 添加到系统服务

  * 使用`pm2 start` （启动服务)
  * 执行`pm2 save` (保存当前已经启动了的服务)
  * 执行`pm2 startup` (设置开机自启的配置)
  * 执行`pm2 startup`以后会得到以下提示 设置环境变量

    ```sh
    [PM2] Init System found: upstart
    [PM2] To setup the Startup Script, copy/paste the following command:
    sudo env PATH=$PATH:/opt/nodejs/bin /opt/nodejs/lib/node_modules/pm2/bin/pm2 startup systemd -u cc --hp /home/cc
    ```

  * 创建系统服务

    ```sh
    sudo env PATH=$PATH:/opt/nodejs/bin /opt/nodejs/lib/node_modules/pm2/bin/pm2 startup systemd -u cc --hp /home/cc
    ```

  * 启动服务

    ```sh
    systemctl enable pm2-cc
    ```