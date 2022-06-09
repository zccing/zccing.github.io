---
title: "etcd安装与配置"
date: 2020-09-06T19:42:47+08:00
lastmod: 2020-10-01T18:54:33+08:00
draft: false
keywords: ["etcd","etcd配置","etcd集群","k8s集群"]
description: "通过二进制方式安装高可用ETCD集群，并采用动态添加一个节点"
tags: ['k8s']
categories: ['容器']
Author: "cc"
autoCollapseToc: true
---

<!--more-->

## 1 节点信息

| IP地址         | 主机名称  | 类型 | 系统    |
| -------------- | --------  | ---- | ------- |
| 192.168.122.10 | node1     | etcd | centos7 |
| 192.168.122.11 | node2     | etcd | centos7 |
| 192.168.122.12 | node3     | etcd | centos8 |


## 2 节点创建etcd账户和防火墙放行

```bash
# 创建用户和目录
groupadd -r etcd
# -k MAIL_DIR=/dev/null参数不创建此用户的mail目录
useradd  -s /sbin/nologin -d /var/lib/etcd -g etcd -c "etcd user" -r -K MAIL_DIR=/dev/null etcd
mkdir -p /opt/etcd/{ssl,etc,log,bin,data}
# 设置权限
chown etcd:etcd -R /opt/etcd/{data,etc,log,ssl}
chmod 700 /opt/etcd/data
restorecon -R /opt/etcd
# 开通防火墙端口
firewall-cmd --add-service=etcd-server --permanent
firewall-cmd --add-service=etcd-client --permanent
firewall-cmd --reload
```

## 3 下载etcd二进制文件

**在所有节点上下载ETCD的二进制文件**

```bash
# etcd的版本
ETCD_VER=v3.4.13
# 通过github下载
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GITHUB_URL}
# 删除旧的程序压缩包，如果有的话
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test
# 解压
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
# 复制etcd执行程序到/opt/etcd/bin目录下
cp /tmp/etcd-download-test/etcd* /opt/etcd/bin/
# 恢复SElinux权限，如果有的话
restorecon -R /opt/etcd
# 清理
rm -rf /tmp/etcd-download-test/
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
```
  
## 4 创建system服务

**在所有节点上都需要创建**

```bash
cat > /usr/lib/systemd/system/etcd.service<<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/opt/etcd/data
EnvironmentFile=/opt/etcd/etc/config
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/opt/etcd/bin/etcd 
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 配置etcd的配置文件位置
echo 'ETCD_CONFIG_FILE="/opt/etcd/etc/etcd.conf"' > /opt/etcd/etc/config
```

## 5 初始集群

**先使用node1和node2创建初始集群，等会在动态添加node3节点**
 
### 5.2 创建配置文件

以下配置文件中选项的官方描述：[https://etcd.io/docs/v3.4.0/op-guide/configuration](https://etcd.io/docs/v3.4.0/op-guide/configuration/)

* node1配置文件

    ```bash
    cat > /opt/etcd/etc/etcd.conf<<EOF
    name: 'node1'
    data-dir: ''
    wal-dir: ''
    # 定义此节点在集群中和其他节点通信的URL
    listen-peer-urls: http://192.168.122.10:2380
    # 定义对外提供服务的URL
    listen-client-urls: http://192.168.122.10:2379,http://127.0.0.1:2379
    # 告诉其他节点，此节点使用哪个URL和你通讯。所以其他集群必须在网络层面可以到达这个地址。URL可以包含域名
    initial-advertise-peer-urls: http://192.168.122.10:2380
    # 告诉集群中其他节点，此节点可以提供服务的URL
    advertise-client-urls: http://192.168.122.10:2379
    # 集群节点之间互相通信的URL，可以动态添加
    initial-cluster: node1=http://192.168.122.10:2380,node3=http://192.168.122.11:2380
    # 加入集群的token
    initial-cluster-token: 'k8s'
    # 初始集群状态，new是新建集群的时候采用，如果是加入已有集群，此处必须是existing
    initial-cluster-state: 'new'
    enable-v2: true
    enable-pprof: true
    debug: false
    logger: zap
    EOF
    ```

* node2配置文件

    ```bash
    cat > /opt/etcd/etc/etcd.conf<<EOF
    name: 'node1'
    data-dir: ''
    wal-dir: ''
    # 定义此节点在集群中和其他节点通信的URL
    listen-peer-urls: http://192.168.122.11:2380
    # 定义对外提供服务的URL
    listen-client-urls: http://192.168.122.11:2379,http://127.0.0.1:2379
    # 告诉其他节点，此节点使用哪个URL和你通讯。所以其他集群必须在网络层面可以到达这个地址。URL可以包含域名
    initial-advertise-peer-urls: http://192.168.122.11:2380
    # 告诉集群中其他节点，此节点可以提供服务的URL
    advertise-client-urls: http://192.168.122.11:2379
    # 集群节点之间互相通信的URL，可以动态添加
    initial-cluster: node1=http://192.168.122.10:2380,node3=http://192.168.122.11:2380
    # 加入集群的token
    initial-cluster-token: 'k8s'
    # 初始集群状态，new是新建集群的时候采用，如果是加入已有集群，此处必须是existing
    initial-cluster-state: 'new'
    enable-v2: true
    enable-pprof: true
    debug: false
    logger: zap
    EOF
    ```

### 5.3 启动ETCD服务

先启动node1，然后在启动node2，每个节点的目录权限别忘记更改

```bash
systemctl daemon-reload
systemctl start etcd.service
systemctl enable etcd.service
```

### 5.4 测试etcd

```bash

# 由于参数太多，所以添加etcdctl参数环境变量
cat >> ~/.bashrc<<EOF
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=http://192.168.122.10:2379,http://192.168.122.11:2379
PATH=$PATH:$HOME/bin:/opt/etcd/bin
export PATH
EOF
source ~/.bashrc
etcdctl endpoint health --write-out=table
```

## 6 动态添加集群成员

前边已经配置了两个成员的etcd集群，etcd集群成员少于三个没有冗余性的，下边我们动态添加一个成员。**在已经添加到etcd集群的服务器（node1）通过etcdctl客户端动态添加成员**

```bash
etcdctl member add node3  --peer-urls=http://192.168.122.12:2380

# 以上命令执行成功会返回新集群的配置参数，如下，记录下来，等会修改集群配置
ETCD_NAME="node3"
ETCD_INITIAL_CLUSTER="node1=http://192.168.122.10:2380,node2=http://192.168.122.11:2380,node3=https://192.168.122.12:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.122.12:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

### 6.1 修改配置文件

根据上边命令回显，node3添加配置如下，参数解析和node1一样

```bash
cat > /opt/etcd/etc/etcd.conf<<EOF
name: $ETCD_NAME
data-dir: ''
wal-dir: ''
listen-peer-urls: http://192.168.122.12:2380
listen-client-urls: http://192.168.122.12:2379,http://127.0.0.1:2379
initial-advertise-peer-urls: $ETCD_INITIAL_ADVERTISE_PEER_URLS
advertise-client-urls: http://192.168.122.12:2379
initial-cluster: $ETCD_INITIAL_CLUSTER
initial-cluster-token: 'k8s'
initial-cluster-state: 'existing'
enable-v2: true
enable-pprof: true
debug: false
logger: zap
EOF
```
        
### 6.2 启动新节点

```bash
# 启动服务
systemctl daemon-reload
systemctl start etcd.service
systemctl enable etcd.service
```
        
### 6.3 测试etcd

```bash
etcdctl endpoint health --write-out=table
```
