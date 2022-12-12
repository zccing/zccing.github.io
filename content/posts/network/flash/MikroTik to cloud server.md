---
title: "云服务器刷MikroTik系统教程"
subtitle: "EIGRP"
date: 2022-08-27T18:43:57+08:00
draft: false
author: "Cc"
authorLink: "www.net-cc.com"
description: "在各大云服务器厂商的现有Ubuntu系统的基础上重新安装MikroTik的CHR版本软路由系统"

tags: ['flash']
categories: ['Route Protocol']

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

在各大云服务器厂商的现有Ubuntu系统的基础上重新安装MikroTik的CHR版本软路由系统

<!--more-->

## 1 前置条件

* 此文档使用的Ubuntu系统版本是16.04或者18.04
* Ubuntu系统只挂载了一个磁盘, 磁盘20G以上空间即可，多了浪费
* 此文档使用的Ubuntu系统的网卡是自动获取IP地址

## 2 下载MikroTik的CHR版本镜像

MikroTik的CHR版本下载页面：https://mikrotik.com/download

```bash
# 更新存储库，并下载wget和unzip程序
apt update  && apt install -y wget unzip
# 下载CHR镜像
wget https://download.mikrotik.com/routeros/7.5/chr-7.5.img.zip
# 解压CHR镜像
gunzip -c chr-7.5.img.zip > chr.img
```

## 3 挂载镜像

```bash
# 安装kpartx
apt-get install kpartx

# 挂载镜像
kpartx -av chr.img
mount -o loop /dev/mapper/loop0p1 /mnt
```

## 4 修改ssh端口

MikroTik系统的默认ssh端口是22，云服务器会根据安全需求，变更ssh端口的，所以我们要更改MikroTik系统镜像的ssh配置

```bash
# 检查设备上的SSH端口
ss -tnlp | grep sshd
```
![ssh端口](/images/network/flash/ssh_port.png "服务器系统配置的ssh端口")

```bash
# 将7272替换成实际的ssh端口
cat >> /mnt/rw/autorun.scr<<"EOF"
/ip service set ssh port=7272 disabled=no
"EOF"
```

## 5 卸载镜像

```bash
umount /mnt
kpartx -dv /dev/loop0
losetup -d /dev/loop0
```

## 6 查看Ubuntu系统的盘符标识

```bash
# 命令
lsblk
# 命令执行后的回显，此回显中的 vda 就是盘符标识
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  25G  0 disk 
└─vda1 252:1    0  25G  0 part /
```
![盘符](/images/network/flash/panfu.png "物理磁盘的盘符")

## 7 Ubuntu系统挂载为只读

```bash
echo u > /proc/sysrq-trigger
```

## 8 替换Ubuntu系统为MikroTik系统

```bash
# 注意替换vda为实际的盘符标识
dd if=chr.img bs=1024 of=/dev/vda
```

## 9 重启系统

```bash
echo s > /proc/sysrq-trigger && sleep 5 && echo b > /proc/sysrq-trigger
```

**至此，云服务器已经是ROS系统了，重新使用admin用户名登陆设备。**