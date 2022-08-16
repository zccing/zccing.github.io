---
title: "libvirt和qemu-kvm安装"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['KVM', 'libvirt','虚拟机','linux虚拟机']
description: "介绍使用libvirt来管理Linux的KVM虚拟机"
tags: ['libvirt']
categories: ['Virtual Machines']
---

记录在linux系统上安装libvirt和qemu-kvm的过程
<!--more-->


## 1 查看是否支持kvm虚拟化

* CPU必需支持虚拟化，可以在/proc/cpuinfo文件中想找flags，如果是inter的显示为vmx，amd的显示为svm `cat /proc/cpuinfo | egrep "(vmx|svm)"`
* CPU必需支持64位操作系统，可以在上述文件中查找lm标记，如果有则支持 `cat /proc/cpuinfo | egrep lm`
* 系统必需为64为的RHEL，且系统版本为RHEL6.4及以上为最佳`uname -a`
* 必需在BIOS里开启CPU的VT功能 `lsmod | grep kvm`

## 2 编译安装qemu和libvirt（未完成）

* 下载文件

    ```bash
    # 可以去官网下载最新的
    wget https://download.qemu.org/qemu-6.2.0.tar.xz
    tar -xf qemu-6.2.0.tar.xz
    cd qemu-6.2.0/
    ````

* 安装需要用到的库文件

    ```bash
    yum install git glib2-devel libfdt-devel pixman-devel zlib-devel bzip2-devel libaio-devel spice-server-devel spice-protocol libusb-devel usbredir-devel
    ```

* 编译安装

    ```bash
    ./configure \
    # 编译完成后安装的目录
    --prefix=/opt/qemu-kvm \
    # 指定数据目录
    --datadir=/home/data/kvm \
    --target-list=i386-softmmu,x86_64-softmmu \
    --enable-system \
    --disable-debug-info \
    --enable-usb-redir \
    --enable-libusb \
    --enable-spice \
    --enable-uuid \
    # 开启KVM支持
    --enable-kvm \
    --enable-bzip2 \
    --enable-linux-aio \
    --enable-tools
    # 编译，并安装到/opt/qemu-kvm目录下
    make -j4 && make install
    ```

* 创建环境变量和添加sytemd服务

## 3 YUM安装

> centos默认存储库的版本过低，添加virt存储库，然后通过yum安装

* 添加qemu扩展存储库

    ```bash
    yum install centos-release-qemu-ev
    ```

* 安装qemu和libvirt

    ```bash
    yum install qemu-kvm libvirt -y
    ```
