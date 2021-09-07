---
title: "etcd安装配置笔记"
date: 2020-09-06T19:42:47+08:00
lastmod: 2020-10-01T18:54:33+08:00
draft: false
keywords: ["etcd","etcd配置","etcd集群","k8s集群"]
description: "通过二进制方式安装最新版的etcd，并做高可用集群配置"
tags: ['k8s','etcd']
categories: ['k8s']
Author: "Net Cc"
autoCollapseToc: true
---

通过二进制方式安装最新版的etcd，并做高可用集群配置
<!--more-->

# 节点信息

| IP地址         | 主机名称 | DNS记录 | 类型 | 系统    |
| -------------- | -------- | ------- | ---- | ------- |
| 192.168.122.10 | node1    | node1   | etcd | centos7 |
| 192.168.122.11 | node2    | node2   | etcd | centos7 |
| 192.168.122.12 | node3    | node3   | etcd | centos8 |


## 节点初始化

```sh
# 创建用户和目录
groupadd -r etcd
# -k MAIL_DIR=/dev/null参数不创建此用户的mail目录
useradd  -s /sbin/nologin -d /var/lib/etcd -g etcd -c "etcd user" -r -K MAIL_DIR=/dev/null etcd
mkdir -p /opt/etcd/{ssl,etc,log,bin,data}
# 开通防火墙端口
firewall-cmd --add-service=etcd-server --permanent
firewall-cmd --add-service=etcd-client --permanent
firewall-cmd --reload
```

# PKI证书管理

**注意：证书我是在node3上进行生成管理的**

使用openssl生成证书，可以参考: [openssl证书管理]({{< relref "openssl.md" >}} "openssl证书管理")

需要这些证书

| 默认 CN     | 父级 CA | O (位于 Subject 中) | 类型           | 主机 (SAN)                                 |
| ----------- | ------- | ------------------- | -------------- | ------------------------------------------ |
| etcd-server | etcd-ca |                     | server, client | <主机名称>, <IP地址>, localhost, 127.0.0.1 |
| etcd-client | etcd-ca | system:masters      | client         |                                            |



## etcd-ca

```sh
mkdir -p ~/ca/etcd-ca/{etcd-server,etcd-client}
cd ~/ca/etcd-ca
openssl genrsa -aes256 -out etcd-ca.key 4096 # 对key使用了aes256进行加密，所以需要输入密码

# etcd-ca的证书
openssl req -extensions v3_ca \
-subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=etcd-ca/emailAddress=admin@admin.com" \
-new -x509 -days 14500 -sha256 \
-key etcd-ca.key -out etcd-ca.crt

# 创建证书签名用到的配置文件
cat > etcd-ca.conf<<EOF
# 用来对服务端证书签名
[ req_server ]
subjectKeyIdentifier = hash
basicConstraints = CA:FALSE
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
subjectAltName = @alt_names
# etcd客户端的IP地址和dns记录（SAN）
[ alt_names ]
DNS.1 = node1
DNS.2 = node2
DNS.3 = node3
DNS.4 = localhost
IP.1 = 192.168.122.10
IP.2 = 192.168.122.12
IP.3 = 192.168.122.13
IP.4 = 127.0.0.1
# 用来对客户端证书签名
[ req_client ]
subjectKeyIdentifier = hash
basicConstraints = CA:FALSE
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF
```

## etcd-server 证书

```sh
cd ~/ca/etcd-ca/etcd-server

# 创建密钥，不要加密，不然程序无法解析
openssl genrsa -out etcd-server.key 2048

# 创建证书请求文件
openssl req -new -sha256 \
-subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=etcd-server/emailAddress=admin@admin.com" \
-key etcd-server.key -out etcd-server.csr

# 使用etcd-ca进行签名获取带有SAN扩展的证书
openssl x509 -in etcd-server.csr -out etcd-server.crt \
-CA ../etcd-ca.crt -CAkey ../etcd-ca.key -CAcreateserial \
-extfile ../etcd-ca.conf -extensions req_server \
-req -sha256 -days 7300

# 更改证书权限
rm -rf etcd-server.csr
chmod 444 etcd-server.crt
chmod 400 etcd-server.key
```

## etcd-client 证书

```sh
cd ~/ca/etcd-ca/etcd-client
# 创建密钥
openssl genrsa -out etcd-client.key 2048

# 创建证书请求文件
openssl req -new -sha256 \
-subj "/C=CN/ST=ShangHai/L=ShangHai/O=system:masters/OU=net-cc.com/CN=etcd-client/emailAddress=admin@admin.com" \
-key etcd-client.key -out etcd-client.csr

# 使用etcd-ca进行签名
openssl x509 -in etcd-client.csr -out etcd-client.crt \
-CA ../etcd-ca.crt -CAkey ../etcd-ca.key -CAcreateserial \
-extfile ../etcd-ca.conf -extensions req_client \
-req -sha256 -days 7300 

rm -f etcd-client.csr
chmod 444 etcd-client.crt
chmod 400 etcd-client.key
```

## 复制证书到各节点

  * etcd-server节点：
    ```sh
    cd ~/ca/etcd-ca/etcd-server
    cp ../etcd-ca.crt etcd-server.crt etcd-server.key /opt/etcd/ssl/
    scp ../etcd-ca.crt etcd-server.crt etcd-server.key root@node1:/opt/etcd/ssl/
    scp ../etcd-ca.crt etcd-server.crt etcd-server.key root@node2:/opt/etcd/ssl/
    ```

  * etcd-client节点
    
    证书密钥文件到需要使用etcd的程序的证书密钥目录内（如K8S），以便访问etcd服务的时候调用即可: `etcd-ca.crt，etcd-client.crt，etcd-client.key`
    
    可以先复制到~/.pki目录，方便etcdctl客户端程序调用

    ```sh
    cd ~/ca/etcd-ca/etcd-client
    mkdir -p ~/.pki
    cp ../etcd-ca.crt etcd-client.crt etcd-client.key ~/.pki/
    ```

# 安装etcd

三个节点上都要先安装etcd，然后在根据节点主机名称、ip地址来修改配置文件

## 下载etcd二进制文件

在node3上下载二进制文件，然后传到其他节点上

```sh
# etcd的版本
ETCD_VER=v3.4.13
# 通过github下载
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GITHUB_URL}
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
# 复制到程序目录
cp /tmp/etcd-download-test/etcd* /opt/etcd/bin/
# 传到其他节点
scp /tmp/etcd-download-test/etcd* root@node1:/opt/etcd/bin/
scp /tmp/etcd-download-test/etcd* root@node2:/opt/etcd/bin/
# 清理
rm -rf /tmp/etcd-download-test/
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
```
  
## 添加system服务

```sh
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

# 复制到node1
scp /usr/lib/systemd/system/etcd.service root@node1:/usr/lib/systemd/system/etcd.service
scp /opt/etcd/etc/config root@node1:/opt/etcd/etc/config
# 复制到node2
scp /usr/lib/systemd/system/etcd.service root@node2:/usr/lib/systemd/system/etcd.service
scp /opt/etcd/etc/config root@node2:/opt/etcd/etc/config
```

# 初始集群

我是先使用node1和node3创建初始集群，然后在动态添加node2节点
 
## 添加配置文件

以下配置文件中选项的官方描述：<https://etcd.io/docs/v3.4.0/op-guide/configuration/>

* node1配置文件

    ```sh
    cat > /opt/etcd/etc/etcd.conf<<EOF
    name: 'node1'
    data-dir: ''
    wal-dir: ''
    listen-peer-urls: https://192.168.122.10:2380
    listen-client-urls: https://192.168.122.10:2379,https://127.0.0.1:2379
    initial-advertise-peer-urls: https://node1:2380
    advertise-client-urls: https://node1:2379
    initial-cluster: node1=https://node1:2380,node3=https://node3:2380
    initial-cluster-token: 'k8s'
    initial-cluster-state: 'new'
    enable-v2: true
    enable-pprof: true
    debug: false
    logger: zap
    # 客户端到服务器端通讯
    client-transport-security:
        cert-file: /opt/etcd/ssl/etcd-server.crt
        key-file: /opt/etcd/ssl/etcd-server.key
        client-cert-auth: true
        trusted-ca-file: /opt/etcd/ssl/etcd-ca.crt
        auto-tls: false
    # 对等通讯 (服务器到服务器/集群)
    peer-transport-security:
        cert-file: /opt/etcd/ssl/etcd-server.crt
        key-file: /opt/etcd/ssl/etcd-server.key
        client-cert-auth: true
        trusted-ca-file: /opt/etcd/ssl/etcd-ca.crt
        auto-tls: false
    EOF
    ```

* node3配置文件

    ```sh
    cat > /opt/etcd/etc/etcd.conf<<EOF
    name: 'node3'
    data-dir: ''
    wal-dir: ''
    listen-peer-urls: https://192.168.122.12:2380
    listen-client-urls: https://192.168.122.12:2379,https://127.0.0.1:2379
    initial-advertise-peer-urls: https://node3:2380
    advertise-client-urls: https://node3:2379
    initial-cluster: node1=https://node1:2380,node3=https://node3:2380
    initial-cluster-token: 'k8s'
    initial-cluster-state: 'new'
    enable-v2: true
    enable-pprof: true
    debug: false
    logger: zap
    # 客户端到服务器端通讯
    client-transport-security:
        cert-file: /opt/etcd/ssl/etcd-server.crt
        key-file: /opt/etcd/ssl/etcd-server.key
        client-cert-auth: true
        trusted-ca-file: /opt/etcd/ssl/etcd-ca.crt
        auto-tls: false
    # 对等通讯 (服务器到服务器/集群)
    peer-transport-security:
        cert-file: /opt/etcd/ssl/etcd-server.crt
        key-file: /opt/etcd/ssl/etcd-server.key
        client-cert-auth: true
        trusted-ca-file: /opt/etcd/ssl/etcd-ca.crt
        auto-tls: false
    EOF
    ```

## 启动ETCD服务

先启动node1，然后在启动node3，每个节点的目录权限别忘记更改

```sh
# 设置权限
chown etcd:etcd -R /opt/etcd/{data,etc,log,ssl}
chmod 700 /opt/etcd/data
restorecon -R /opt/etcd

# 启动服务
systemctl daemon-reload
systemctl start etcd.service
systemctl enable etcd.service
```

## 测试etcd

```sh

# 由于参数太多，所以添加etcdctl参数环境变量
cat >> ~/.bashrc<<EOF
export ETCDCTL_API=3
export ETCDCTL_CACERT=$HOME/.pki/etcd-ca.crt
export ETCDCTL_CERT=$HOME/.pki/etcd-client.crt
export ETCDCTL_KEY=$HOME/.pki/etcd-client.key
export ETCDCTL_ENDPOINTS=https://node1:2379,https://node3:2379,https://node2:2379

PATH=$PATH:$HOME/bin:/opt/etcd/bin
export PATH
EOF
source ~/.bashrc
etcdctl endpoint health --write-out=table
```

# 动态添加集群成员

前边已经配置了两个成员的etcd集群，etcd集群成员少于三个没有冗余性的，下边我们动态添加一个成员

## 添加集群成员

在已经安装etcd集群的节点（node3）通过etcdctl客户端动态添加节点

```sh
etcdctl member add node2  --peer-urls=https://node2:2380

# 以上命令执行成功会返回新集群的配置参数，如下，记录下来，等会修改集群配置
ETCD_NAME="node2"
ETCD_INITIAL_CLUSTER="node1=https://node1:2380,node2=https://node2:2380,node3=https://node3:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://node2:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

## 修改配置文件

> 根据上边命令回显，node2添加配置如下

```sh
cat > /opt/etcd/etc/etcd.conf<<EOF
name: 'node2'
data-dir: ''
wal-dir: ''
listen-peer-urls: https://192.168.122.11:2380
listen-client-urls: https://192.168.122.11:2379,https://127.0.0.1:2379
initial-advertise-peer-urls: https://node2:2380
advertise-client-urls: https://node2:2379
initial-cluster: node1=https://node1:2380,node2=https://node2:2380,node3=https://node3:2380
initial-cluster-token: 'k8s'
initial-cluster-state: 'existing'
enable-v2: true
enable-pprof: true
debug: false
logger: zap
client-transport-security:
    cert-file: /opt/etcd/ssl/etcd-server.crt
    key-file: /opt/etcd/ssl/etcd-server.key
    client-cert-auth: true
    trusted-ca-file: /opt/etcd/ssl/etcd-ca.crt
    auto-tls: false
peer-transport-security:
    cert-file: /opt/etcd/ssl/etcd-server.crt
    key-file: /opt/etcd/ssl/etcd-server.key
    client-cert-auth: true
    trusted-ca-file: /opt/etcd/ssl/etcd-ca.crt
    auto-tls: false
EOF
```
        
## 启动新节点

```sh
# 设置目录用户组
chown etcd:etcd -R /opt/etcd
chmod 700 /opt/etcd/data
restorecon -R /opt/etcd

# 启动服务
systemctl daemon-reload
systemctl start etcd.service
systemctl enable etcd.service
```
        
## 测试etcd

```sh
etcdctl endpoint health --write-out=table
```
