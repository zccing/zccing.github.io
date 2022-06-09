---
title: "shadowsocks"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-07T18:51:22+08:00
draft: false
keywords: ['shadowsocks', 'ss','小飞机']
description: "通过二进制方式安装ss，用来做国际优化"
tags: ['vpn']
categories: ['网络技术']
---

通过二进制方式安装ss，在阿里云香港购买一台服务器，用来做国际优化
<!--more-->

## 安装shadowsocks-libev服务端

* 存储库安装

    ```bash
    cd /etc/yum.repos.d/
    wget https://copr.fedorainfracloud.org/coprs/librehat/shadowsocks/repo/epel-7/librehat-shadowsocks-epel-7.repo
    yum check
    yum install epel-release -y
    yum install shadowsocks-libev -y
    ```

* 编译安装
  * 安装需要的lib库

    ```bash
    yum install epel-release -y
    yum install gcc make pcre-devel mbedtls-devel libsodium-devel c-ares-devel libev-devel libnetfilter_conntrack-devel libnetfilter_conntrack -y
    # 如果要安装文档的还需要以下几个依赖
    yum install gettext autoconf libtool automeke xmlto -y
    ```

  * 下载软件，解压缩

    ```bash
    cd /usr/local/src
    wget https://github.com/shadowsocks/shadowsocks-libev/releases/download/v3.2.3/shadowsocks-libev-3.2.3.tar.gz
    tar -xvf shadowsocks-libev-3.2.3.tar.gz
    cd shadowsocks-libev-3.2.3
    mkdir /opt/shadowsocks-libev
    ```

  * 编译安装

    ```bash
    ./configure --prefix=/opt/shadowsocks-libev/ \
    --disable-documentation
    make
    make install
    ```

  * 添加系统服务

    ```bash
    cd /usr/local/src/shadowsocks-libev-3.2.3
    cp rpm/SOURCES/systemd/shadowsocks-libev.service /usr/lib/systemd/system/
    cp rpm/SOURCES/systemd/shadowsocks-libev.default /etc/sysconfig/shadowsocks-libev

    # 编辑服务文件，修改`/usr/bin/`为`/opt/shadowsocks-libev/`
    sed -i 's/\/usr\/bin\//\/opt\/shadowsocks-libev\/bin\//g' /usr/lib/systemd/system/shadowsocks-libev.service

    # 开机启动服务
    systemctl daemon-reload
    systemctl enable shadowsocks-libev.service

    # 这里启动服务会失败，因为没有配置文件
    systemctl start shadowsocks-libev.service
    ```

* docker安装
    > 参考`docker hub`上的介绍

* 配置
  * 编辑配置文件

    ```bash
    # 添加配置文件
    cat > /etc/shadowsocks-libev/config.json <<EOF
    {
      "server":["[::0]","0.0.0.0"],
      "server_port":19438,
      "local_port":1081,
      "password":"cc951021\$ps",
      "timeout":60,
      "method":"chacha20-ietf-poly1305"
    }
    EOF
    ```

  * 优化
    > 参照这一篇文章进行优化[shadowsocks advanced config](https://shadowsocks.org/en/config/advanced.html)

    ```bash
    cat > /etc/sysctl.d/98-shadowsocks.conf <<EOF
    # max open files
    fs.file-max = 51200
    # max read buffer
    net.core.rmem_max = 67108864
    # max write buffer
    net.core.wmem_max = 67108864
    # default read buffer
    net.core.rmem_default = 65536
    # default write buffer
    net.core.wmem_default = 65536
    # max processor input queue
    net.core.netdev_max_backlog = 4096
    # max backlog
    net.core.somaxconn = 4096
    # resist SYN flood attacks
    net.ipv4.tcp_syncookies = 1
    # reuse timewait sockets when safe
    net.ipv4.tcp_tw_reuse = 1
    # turn off fast timewait sockets recycling
    net.ipv4.tcp_tw_recycle = 0
    # short FIN timeout
    net.ipv4.tcp_fin_timeout = 30
    # short keepalive time
    net.ipv4.tcp_keepalive_time = 1200
    # outbound port range
    net.ipv4.ip_local_port_range = 10000 65000
    # max SYN backlog
    net.ipv4.tcp_max_syn_backlog = 4096
    # max timewait sockets held by system simultaneously
    net.ipv4.tcp_max_tw_buckets = 5000
    # turn on TCP Fast Open on both client and server side
    net.ipv4.tcp_fastopen = 3
    # TCP receive buffer
    net.ipv4.tcp_rmem = 4096 87380 67108864
    # TCP write buffer
    net.ipv4.tcp_wmem = 4096 65536 67108864
    # turn on path MTU discovery
    net.ipv4.tcp_mtu_probing = 1
    # for high-latency network
    net.ipv4.tcp_congestion_control = hybla
    # for low-latency network, use cubic instead
    # net.ipv4.tcp_congestion_control = cubic
    EOF


    sysctl -p /etc/sysctl.d/98-shadowsocks.conf
    ```

## 安装shadowsocks客户端

* 安卓
  > google play商店下载
  > [备用下载地址](https://github.com/shadowsocks/shadowsocks-android/releases)
* ios
  > 商店下载
* linux
    > linux的安装步骤和安装服务端一样
    > 也可以安装`shadowsocks-qt5`版本，不过我感觉`libev`版本配合`chrome`的`SwitchyOmega`插件挺好用的

  * 安装后的配置

    ```bash
    cat > /etc/shadowsocks-libev/config.json <<EOF
    {
      "server":["service-ip"],
      "server_port":server-port,
      "local_port":1081,
      "password":"server-passwd",
      "timeout":60,
      "method":"server加密算法"
    }
    ```

  * 添加系统服务

    ```bash
    cp rpm/SOURCES/systemd/shadowsocks-libev-local.service /usr/lib/systemd/system/
    sed -i 's/\/usr\/bin\//\/opt\/shadowsocks-libev\/bin\//g' /usr/lib/systemd/system/shadowsocks-libev-local.service
    # 开机启动服务
    systemctl daemon-reload
    systemctl enable shadowsocks-libev-local.service
    systemctl start shadowsocks-libev-local.servic
    ```

* windows
  > [下载地址](https://github.com/shadowsocks/shadowsocks-windows/releases)
* macOS
  > [下载地址](https://github.com/shadowsocks/ShadowsocksX-NG/releases)

## 安装shadowsocks-manager

> `shadowsocks-manager`是`shadowsocks-libev`的管理工具，需要先安装`shadowsocks-libev`服务端然后启用`shadowsocks-libev`的`API`和`shadowsocks-manager`对接

* 安装`nodejs`

我就不造轮子了，上大招[官方链接](https://nodejs.org/zh-cn/download/)

* 安装`sqlite`

    > `sqlite`一般都自带的无需安装

    ```bash
    yum install sqlite
    ```

* 安装`shadowsocks-manager`

  * 拓扑

    ```bash
    +-------------+    +-------------+       +------+
    | Shadowsocks |    | Shadowsocks |  ...  |      |
    | manager API |    | manager API |       |      |
    +-------------+    +-------------+       +------+
          |                 |                  |
          |                 |                  |
    +-------------+    +-------------+       +------+
    | ssmgr       |    | ssmgr       |  ...  |      |
    | with type s |    | with type s |       |      |
    +-------------+    +-------------+       +------+
          |                 |                  |
          +------------+----+--------  ...  ---+
                        |
                        |
                +---------------+
                | ssmgr plugins |
                |  with type m  |
                +---------------+
    ```

  * 从源代码安装

    ```bash
    git clone https://github.com/shadowsocks/shadowsocks-manager.git
    cd shadowsocks-manager
    npm i -g
    ```

  * 从NPM安装

    > 升级前请做好备份,请勿跨版本升级，例如`0.21.0`可以升级到`0.22.x`，但不能直接升级到`0.23.x`
    > 通过NPM安装的可执行文件(ssmgr):`/opt/nodejs/bin`,程序文件:`ib/node_modules/shadowsocks-manager/`

    ```bash
    npm i -g shadowsocks-manager
    or
    npm i -g shadowsocks-manager --unsafe-perm
    # 升级 a.b.c为版本号
    npm i -g shadowsocks-manager@a.b.c
    ```

* 配置`shadowsocks-manager`
  * 创建配置文件

    ```bash
    cat > $HOME/.ssmgr/default.yml <<EOF
    type: s
    shadowsocks:
      address: 127.0.0.1:6001
    manager:
      address: webgui的ip地址:59418
      password: '951021@cc'
    db: 'db.sqlite'
    EOF
    ```

  * 启动节点

    ```bash
    pm2 --name "node" -f start ssmgr -x -- -c $HOME/.ssmgr/default.yml -r libev:chacha20-ietf-poly1305
    ```

* 配置`webgui`

  > 每个节点都需要按照以上进行配置，web只需要在一个节点上配置就好了，简单来说是`web页面`利用`shadowsocks-manager`的`webgui`扩展通过每个节点的`ssmgr`('shadowsocks-manager')程序调用`shadowsocks`的`API`控制`shadowsocks`的

  * 创建配置文件

    > 在`$HOME/.ssmgr`目录下创建一个`web.yml`文件，内容如下

    ```yml
    type: m
    manager:
      address: 127.0.0.1:59418
      password: '951021@cc'
      # 这部分的端口和密码需要跟上一步 manager 参数里的保持一致，以连接 type s 部分监听的 tcp 端口
    plugins:
      flowSaver:
        use: true
      user:
        use: true
      account:
        use: true
      macAccount:
        use: true
      group:
        use: true
      email:
        use: true
        type: 'smtp'
        username: 'username'
        password: 'password'
        host: 'smtp.your-email.com'
        # 这部分的邮箱和密码是用于发送注册验证邮件，重置密码邮件
      webgui:
        use: true
        host: '0.0.0.0'
        port: '80'
        site: 'http://yourwebsite.com'
        # cdn: 'http://xxx.com' # 静态资源cdn地址，可省略
        # icon: 'icon.png' # 自定义首页图标，默认路径在 ~/.ssmgr 可省略
        # skin: 'default' # 首页皮肤，可省略
        # googleAnalytics: 'UA-xxxxxxxx-x' # Google Analytics ID，可省略
        gcmSenderId: '456102641793'
        gcmAPIKey: 'AAAAGzzdqrE:XXXXXXXXXXXXXX'
      webgui_telegram: // telegram 机器人的配置，可省略
        use: true
        token: '191374681:AAw6oaVPR4nnY7T4CtW78QX-Xy2Q5WD3wmZ'
      alipay:
        # 如果不使用支付宝，这段可以去掉
        use: true
        appid: 2015012108272442
        notifyUrl: 'http://yourwebsite.com/api/user/alipay/callback'
        merchantPrivateKey: 'xxxxxxxxxxxx'
        alipayPublicKey: 'xxxxxxxxxxx'
        gatewayUrl: 'https://openapi.alipay.com/gateway.do'
      paypal:
        # 如果不使用paypal，这段可以去掉
        use: true
        mode: 'live' # sandbox or live
        client_id: 'At9xcGd1t5L6OrICKNnp2g9'
        client_secret: 'EP40s6pQAZmqp_G_nrU9kKY4XaZph'

    db: 'webgui.sqlite'
    ```

  * 启动`web`

    ```bash
    pm2 --name "web" -f start ssmgr -x -- -c $HOME/.ssmgr/web.yml
    ```

## 安装配置KCPtun

> 加速tcp传输，但会造成双倍流量，不建议，流量贵～～～
