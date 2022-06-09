---
title: "openssh配置指南"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-05T23:11+08:00
draft: false
keywords: ['linux免密登录', 'openssh','linux密钥登录']
description: "openssh配置指南，linux免密登录配置方法等等"
tags: ['linux']
categories: ['linux']
---

openssh配置指南，linux免密登录配置方法等等
<!--more-->

# openssh学习笔记<!-- omit in toc -->

## CentOS 7 OpenSSH 配置

默认自动安装SSH并开启密码登陆
开启密钥登陆

```bashell
ssh localhost
# 首次登陆下,生成~/.ssh目录
exit
# 退出刚才的 ssh localhost
cd ~/.ssh/
# 若没有该目录，请先执行一次ssh localhost
ssh-keygen -t rsa
# 会有提示，都按回车就可以
cat id_rsa.pub >> authorized_keys
# 加入授权
chmod 600 ./authorized_keys# 修改文件权限
vim /etc/ssh/sshd_config
# 修改sshd_config，并把PubkeyAuthentication yes前边的#去除
下载id_rsa  id_rsa.pub两个密钥
# CRT下载方式为 Alt+P建进入SFTP,get命令下载，help命令帮助，要SSH登陆
```

sshd_config配置
> service sshd restart# 修改配置文件后重启SSHDf服务

```ini
UseDNS=no
# DNS反向解析，根据用户的IP使用反向DNS找到主机名，再使用DNS找到IP最后匹配一下登录的IP是否合法。
```

---------------**以上设置影响SSH登陆速度**---------------