---
title: "编译安装git"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['git', '安装git','编译安装git','linux']
description: "使用编译的方式在linux服务器上安装git"
tags: ['git']
categories: ['linux software']
---

使用编译的方式在linux服务器上安装git
<!--more-->
# git学习笔记

## 1 配置编译环境

### 1.1 下载源代码包

```bashell
mkdir ~/src
cd ~/src
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.19.1.tar.gz
```

### 1.2 安装依赖库文件

```bashell
yum install curl-devel expat-devel openssl-devel zlib-devel asciidoc
```

## 2 编译配置

### 2.1 编译

```bashell
mkdir /opt/git
make prefix=/opt/git/ all
sudo make prefix=/opt/git/ install
echo $? # 查看是否编译安装完成
```

### 2.2 设置环境变量

```bashell
mkdir /opt/bin
ln -s /opt/git/bin/* /opt/bin
echo "export PATH=$PATH:/opt/bin/" >> /etc/profile
source /etc/profile
```

### 2.3 测试

```bashell
git --version

# 初始化
git config --global user.email "alex_zjf@163.com"
git config --global user.name "cc"
git config --global user.editor "vim"
```