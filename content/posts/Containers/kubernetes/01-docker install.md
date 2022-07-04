---
title: "centos安装docker"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-08-30T01:37:56+08:00
draft: false
keywords: ['docker', 'linux', '容器']
description: "使用centos系统如何安装docker容器，采用的是通过yum源方式进行安装，主要为了方便快速"
tags: ['kubernetes']
categories: ['containers']
---

<!--more-->

## 1 开启网络转发和PNAT

> centos6 参考这里：<https://blog.csdn.net/qianye_111/article/details/78987161>

```bash
firewall-cmd --add-masquerade --permanent --zone=public # 更改防火墙开启PNAT
firewall-cmd --reload
# 开启转发
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
cat /proc/sys/net/ipv4/ip_forward  # 查看是否开启了ip转发，如果返回1表示开启了
```

## 2 通过yum安装docker-ce

使用仓库安装，方便，也可以二进制安装，编译的话不建议了，时间就是金钱，我的朋友。[阿里云镜像站介绍](<https://yq.aliyun.com/articles/110806>)

```bash
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo systemctl docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，你可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
# 将 [docker-ce-test] 下方的 enabled=0 修改为 enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2 : 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

## 3 配置docker网络

* 如果要docker内容器要固定ip地址，需要创建一个network

    ```bash
    # 把ip改为相应的即可
    docker network create --attachable --driver bridge --gateway 192.168.243.1 --subnet 192.168.243.1/24 --ipam-driver default mybridge
    ```

* 不同主机之间的docker相互访问，首先要确定主机与主机之间是否互联，不能经过nat，然后在主机配置到docker网段的路由即可

    ```bash
    # linux配置路由，注意替换ip地址
    # centos7
    firewall-cmd --add-masquerade --permanent --zone=public
    firewall-cmd --reload
    ip route add 192.168.240.0/20 via 192.168.250.4
    ip route del 192.168.240.0/20 via 192.168.250.4 # 删除路由
    # centos 6
    sysctl -w net.ipv4.ip_forward=1
    route add -net 10.20.30.48 netmask 255.255.255.248 gw 10.20.30.41
    or
    route add -net 192.168.1.0/24 eth1
    route del -net 10.20.30.40 netmask 255.255.255.248 eth0 # 删除路由

    # windows配置路由
    route -p add 157.0.0.0 MASK 255.0.0.0 157.55.80.1 METRIC 3 IF 2
    # 修改删除路由
    route CHANGE 157.0.0.0 MASK 255.0.0.0 157.55.80.5 METRIC 2 IF 2 # 修改
    route delete 157.0.0.0 # 删除
    ```

## 4 docker配置ip、log和存储驱动

```bash
mkdir -p /etc/docker
# 创建配置文件，配置docker默认网桥的IP地址为192.168.66.1/24，使用163和中国科技大学的镜像站
cat > /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "http://hub-mirror.c.163.com"
    ],
    "exec-opts": [
        "native.cgroupdriver=systemd"
    ],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    },
    "bip": "192.168.66.1/24",
    "ipv6": false,
    "iptables": false,
    "ip-masq": false,
    "dns": [
        "223.5.5.5",
        "223.6.6.6"
    ],
    "dns-search":[
        "net-cc.com"
    ],
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true"
    ]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
```

## 5 docker daemon.json配置文件格式

[官方的完整配置](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)样式，需要根据你的需要修改

```json
{
  "allow-nondistributable-artifacts": [],
  "api-cors-header": "",
  "authorization-plugins": [],
  "bip": "",
  "bridge": "",
  "cgroup-parent": "",
  "cluster-advertise": "",
  "cluster-store": "",
  "cluster-store-opts": {},
  "containerd": "/run/containerd/containerd.sock",
  "containerd-namespace": "docker",
  "containerd-plugin-namespace": "docker-plugins",
  "data-root": "",
  "debug": true,
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    },
    {
      "base": "172.31.0.0/16",
      "size": 24
    }
  ],
  "default-cgroupns-mode": "private",
  "default-gateway": "",
  "default-gateway-v6": "",
  "default-runtime": "runc",
  "default-shm-size": "64M",
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  },
  "dns": [],
  "dns-opts": [],
  "dns-search": [],
  "exec-opts": [],
  "exec-root": "",
  "experimental": false,
  "features": {},
  "fixed-cidr": "",
  "fixed-cidr-v6": "",
  "group": "",
  "hosts": [],
  "icc": false,
  "init": false,
  "init-path": "/usr/libexec/docker-init",
  "insecure-registries": [],
  "ip": "0.0.0.0",
  "ip-forward": false,
  "ip-masq": false,
  "iptables": false,
  "ip6tables": false,
  "ipv6": false,
  "labels": [],
  "live-restore": true,
  "log-driver": "json-file",
  "log-level": "",
  "log-opts": {
    "cache-disabled": "false",
    "cache-max-file": "5",
    "cache-max-size": "20m",
    "cache-compress": "true",
    "env": "os,customer",
    "labels": "somelabel",
    "max-file": "5",
    "max-size": "10m"
  },
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5,
  "max-download-attempts": 5,
  "mtu": 0,
  "no-new-privileges": false,
  "node-generic-resources": [
    "NVIDIA-GPU=UUID1",
    "NVIDIA-GPU=UUID2"
  ],
  "oom-score-adjust": -500,
  "pidfile": "",
  "raw-logs": false,
  "registry-mirrors": [],
  "runtimes": {
    "cc-runtime": {
      "path": "/usr/bin/cc-runtime"
    },
    "custom": {
      "path": "/usr/local/bin/my-runc-replacement",
      "runtimeArgs": [
        "--debug"
      ]
    }
  },
  "seccomp-profile": "",
  "selinux-enabled": false,
  "shutdown-timeout": 15,
  "storage-driver": "",
  "storage-opts": [],
  "swarm-default-advertise-addr": "",
  "tls": true,
  "tlscacert": "",
  "tlscert": "",
  "tlskey": "",
  "tlsverify": true,
  "userland-proxy": false,
  "userland-proxy-path": "/usr/libexec/docker-proxy",
  "userns-remap": ""
}
```