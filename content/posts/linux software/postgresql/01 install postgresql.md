---
title: "Debian11安装PostgreSQL13"
date: 2022-03-05T19:50:37+08:00
draft: false
keywords: []
description: ""
tags: ['PostgreSQL']
categories: ['linux software']
author: ""
autoCollapseToc: false
---

在debian 11系统上安装PostgreSQL13, 也可以通过本文档了解如何在其他linux发行版上安装PostgreSQL13
<!--more-->

## 1 更新debian11的镜像源

debian11默认的镜像源是国外的，下载比较慢，替换为国内的镜像源，这里用的是阿里云镜像源

```bash
# 优化内核参数
cat > /etc/sysctl.d/PostgreSQL.conf<<EOF
vm.overcommit_memory=2
EOF
sysctl -p /etc/sysctl.d/PostgreSQL.conf
# 备份原来的镜像源
mv /etc/apt/sources.list /etc/apt/sources.list.bak
# 更新新的阿里云镜像源
cat > /etc/apt/sources.list <<EOF
deb http://mirrors.aliyun.com/debian/ bullseye main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye main non-free contrib
deb http://mirrors.aliyun.com/debian-security/ bullseye-security main
deb-src http://mirrors.aliyun.com/debian-security/ bullseye-security main
deb http://mirrors.aliyun.com/debian/ bullseye-updates main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-updates main non-free contrib 
deb http://mirrors.aliyun.com/debian/ bullseye-backports main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-backports main non-free contrib
EOF
```

## 2 添加PostgreSQL存储库

```bash
# 安装一些必要的工具
apt update && apt install -y gnupg2 wget
# 创建存储库
sudo sh -c 'echo "deb  [arch=amd64] https://mirrors.aliyun.com/postgresql/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
# 添加存储库的密钥
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
# 更新存储库
apt update
```

## 3 下载PostgreSQL

```bash
# 安装指定版本,后续的配置都是基于此版本的， 如果要安装最新版的话，命令是`apt install PostgreSQL`
apt install PostgreSQL-13 -y
# 将PostgreSQL添加到开机启动
systemctl enable PostgreSQL
```

## 4 配置数据目录

正常情况应该将数据存储到一个单独的磁盘中，和系统磁盘区分开；

我的环境中将`/data`目录挂载到一个单独的磁盘中，所以将数据保存到`/data/pgsql`目录中

* 创建目录，并设置权限

    ```bash
    # 创建目录
    mkdir /data/pgsql
    # 设置此目录的权限
    chown postgres:postgres -R /data/pgsql
    ```

* 变更数据目录

    > PostgreSQL支持一台机器多实例配置（相当于一台服务器多个虚拟机），我们通过删除默认实例，重新创建一个实例并指定实例的数据目录

    ```bash
    # 停止并删除默认安装创建的实例
    pg_dropcluster --stop 13 main 
    # 在`/data/pgsql`目录重新创建PostgreSQL实例，并初始化实例
    pg_createcluster -d /data/pgsql 13 main 
    # 启动新创建的PostgreSQL实例
    pg_ctlcluster 13 main start
    ```
* 检查数据目录

    ```bash
    # 切换到postgres用户，使用这个用户可以免密登录到pgsql
    su - postgres
    # 检查数据目录, 返回的结果中告诉了我们当前目录是`/data/pgsql`
    psql -c 'SHOW data_directory;'
    ```

## 5 修改默认PostgreSQL的密码

PostgreSQL安装成功之后，会默认创建一个名为postgres的Linux用户

初始化PostgreSQL实例后，实例数据库中会创建一名超级用户`postgres`和一个名为`postgres`的数据库

`postgres`的数据库中存储`PostgreSQL`的基础信息，例如用户信息等等，相当于MySQL中默认的名为mysql数据库

为了方便我们使用postgres账号进行管理和安全考虑，我们可以修改数据库和linux用户的账号的密码

* 修改linux用户`postgres`的密码

    ```bash
    passwd postgres # 修改linux用户的密码，如果不明白，后边的也不要看了，看也看不懂
    ```

* 修改实例数据库的超级用户`postgres`的密码
    > 如果linux用户的名称和超级用户的名称相同，登录数据库是不需要密码的，这个权限是在`/etc/PostgreSQL/13/main/pg_hba.conf`中配置，后边会讲到此文件的用法

    ```bash
    # 切换到postgres用户，并修改数据库超级用户的密码， ‘NewPassword’替换为你的密码即可
    su - postgres
    psql -c "ALTER USER postgres WITH PASSWORD 'NewPassword';"

    exit  # 退出postgres用户，切换到root用户
    psql -U postgres -h localhost # 输入刚刚更改的密码尝试连接
    ```

## 6 客户端连接认证

客户端认证是由一个`/etc/PostgreSQL/13/main/pg_hba.conf`配置文件控制（HBA表示基于主机的认证）；此文件在初始化数据目录时生成。

`pg_hba.conf`文件的每一行是一条记录，空白行和`#`注释字符后面的任何文本都将被忽略。记录不能跨行。一条记录由空格或`table制表符`分隔的域组成。在数据库、用户或地址域中 关键字`all`、`replication`等有特殊含义，并且只是匹配一个有该名字的数据库、用户或主机。[官方](http://www.postgres.cn/docs/13/auth-pg-hba-conf.html)

* 允许所有`169.3.248.0/21`网段的所有用户通过密码访问数据库

    ```
    cat >> /etc/PostgreSQL/13/main/pg_hba.conf <<EOF
    host    all             all             169.3.248.0/21          md5
    EOF
    ```

## 7 安装timescale扩展

PostgreSQL安装timescale扩展就可以变成一个时序数据库了,新的名字是`timescaledb`, 只是改变了存储和索引不影响PostgreSQL原有的主从架构和使用方式

* 添加存储库

    ```bash
    # 设置timescale的存储库
    apt install gnupg PostgreSQL-common apt-transport-https lsb-release wget
    sh -c "echo 'deb https://packagecloud.io/timescale/timescaledb/debian/ $(lsb_release -c -s) main' > /etc/apt/sources.list.d/timescaledb.list"

    # 添加timescale的存储库密钥
    curl -L https://packagecloud.io/timescale/timescaledb/gpgkey | sudo apt-key add -
    ```

* 下载并安装timescale

    ```bash
    apt install timescaledb-2-PostgreSQL-13
    ```

* 配置PostgreSQL支持timescale扩展

    ```bash
    # 修改/etc/PostgreSQL/13/main/PostgreSQL.conf配置文件中的shared_preload_libraries参数，并去掉注释使其生效
    vim /etc/postgresql/13/main/postgresql.conf

    shared_preload_libraries = 'timescaledb'  # 不区分大小写

    # 修改好了以后重启服务，使其加载此扩展
    systemctl restart postgresql
    ```

* 为数据库安装timescale扩展

    ```sql
    # 切换到postgres用户
    su - postgres
    # 登录数据库
    psql
    
    # 为所有已存在的数据库安装扩展, 由于其他数据库不支持timescaledb,所以我没用这个命令
    CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

    # 创建一个测试数据库
    CREATE DATABASE timeseries_test;
    # 查看数据库
    \l
    # 进入数据库
    \c timeseries_test
    # 为当前进入的数据安装扩展,不影响其他数据库
    CREATE EXTENSION timescaledb;
    # 检查扩展
    \dx
    SELECT default_version, installed_version FROM pg_available_extensions WHERE name = 'timescaledb';
    # 切换数据库
    \c PostgreSQL
    # 删除测试数据库
    DROP DATABASE timeseries_test;
    ```

## 8 PostgreSQL参数优化

整理了一些PostgreSQL参数的优化，仅供参考，请依照实际环境为准

```ini
# 我的PostgreSQL的实例配置文件(默认实例):/etc/PostgreSQL/13/main/PostgreSQL.conf
# 新增或修改下列属性设置（使用命令“/”来查找）

# （修改）监听所有IP
listen_addresses = '*'
# （修改）最大连接数，超过的话建议程序设置连接池减少开销
max_connections = 1000
# (修改)缓冲数据到内存中，建议设置物理内存的1/4，根据服务器负载调整设置
shared_buffers = 8GB 
# （修改）主要用于Postgre查询优化器,建议设置物理内存的3/4,较高的值更有可能使用索引扫描，较低的值则有可能使用顺序扫描
effective_cache_size = 24GB 
# （修改）用于写入临时文件之前内部排序操作和散列表使用的内存量，增加work_mem参数将使PostgreSQL可以进行更大的内存排序，和max_connections有关，连接越多占用内存越多，合理分配
# Total RAM * 0.25 / max_connections
work_mem = 64M
# (修改)指定维护操作使用的最大内存量，例如（Vacuum、Create Index和Alter Table Add Foreign Key），默认值是64MB。由于通常正常运行的数据库中不会有大量并发的此类操作，可以设置的较大一些，提高清理和创建索引外键的速度。
# Total RAM * 0.05
maintenance_work_mem = 2GB
```

推荐一个工具[PostgreSQLtuner](https://github.com/jfcoz/PostgreSQLtuner)，使用此工具来优化参数

```bash
# 下载
wget https://raw.githubusercontent.com/jfcoz/PostgreSQLtuner/master/PostgreSQLtuner.pl
chmod u+x PostgreSQLtuner.pl

# 登录数据库，创建testdb库
psql -U postgres -h localhost -c 'CREATE DATABASE testdb;'

# 运行命令
PostgreSQLtuner.pl --host=localhost --database=testdb --user=username --password=qwerty

# 删除testdb库
psql -U postgres -h localhost -c 'DROP DATABASE testdb;'

# 根据结果调优自己的服务器和数据库吧
```