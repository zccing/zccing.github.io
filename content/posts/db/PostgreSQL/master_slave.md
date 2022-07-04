---
title: "Master_slave"
date: 2022-03-05T19:51:22+08:00
draft: false
keywords: []
description: ""
tags: ['postgresql']
categories: ['数据库']
author: "Cc"
autoCollapseToc: false
---

配置PostgreSQL13主从复制(异步流复制模式), 此文章的前提是已经安装好了两台postgresql数据库服务器

<!--more-->

## 1 环境介绍

**我的环境中有两台数据库服务器；主数据库服务器IP:169.3.250.215，备数据库服务器IP:169.3.254.215**

## 2 创建同步账号

只需要在主数据库节点创建，从节点不需要，等会要从主节点同步所有信息的

```sql
psql -U postgres -h localhost
# 同步账号
create user repl  REPLICATION  LOGIN ENCRYPTED  PASSWORD 'XXXXXX';

# 心跳账号，只要能登录就好
create role nobody login encrypted password 'xxxxxxx';

# 创建一个业务账号和业务数据库
create role zabbix login encrypted password 'xxxxxxx';
create database zabbix owner zabbix; 
# 为业务数据库加载timescale扩展, 可选,前提是安装了这个扩展
CREATE EXTENSION timescaledb;
```

## 3 策略放行

我这里放行了所有，只要是来自`169.3.248.0/21`网段的主机，都可以进行账号密码认证

```bash
# 放行所有
cat >> /etc/postgresql/13/main/pg_hba.conf<<EOF
host    replication     all             169.3.248.0/21          md5
EOF
```


## 4 配置修改

这里主要修改用户主从同步的参数，主从服务器都要设置, 万一那天从服务器提升为主了呢,

```bash
vim /etc/postgresql/13/main/postgresql.conf

# 新增或修改下列属性设置（使用命令“/”来查找）
archive_mode = off # （修改）关闭归档, 归档是定时恢复用的，流复制不是必须的
wal_level = replica # （添加）
max_wal_senders = 20 # (修改) 最多有几个流复制连接
wal_sender_timeout = 60s # （修改）设置流复制主机发送数据的超时时间
max_replication_slots = 10 # (修改) 设置支持的复制槽数量
max_slot_wal_keep_size = 1G # (修改) 设置复制槽保留的wal最大大小,默认单位是M
hot_standby = on # (修改) 说明这台机器不仅仅是用于数据归档，也用于数据查询
```

## 5 从节点同步主节点的数据库

```bash
# 同步数据
pg_basebackup -h 169.3.250.215 -U repl -p 5432 -F p  -X stream -v -P -R -D /data/pgsql -C -S slave01 -l slave01
chown -R postgres:postgres -R /data/pgsql
# 在从节点检查状态，查询结果为"f"表示主库, 't'表示从库
psql -U postgres -h localhost -c "select pg_is_in_recovery();"
# 在主节点检查同步状态
psql -U postgres -h localhost -c "select * from pg_stat_replication;" -d postgres
```

### `pg_basebackup`参数解析

```bash
pg_basebackup 在运行的PostgreSQL服务器上执行基础备份.

使用方法:
  pg_basebackup [选项]...

控制输出的选项:
  -D, --pgdata=DIRECTORY 接收基础备份到指定目录
  -F, --format=p|t       输出格式 (纯文本 (缺省值), tar压缩格式)
  -r, --max-rate=RATE    传输数据目录的最大传输速率
                         (单位 kB/s, 也可以使用后缀"k" 或 "M")
  -R, --write-recovery-conf
                         为复制写配置文件
  -T, --tablespace-mapping=OLDDIR=NEWDIR
                         将表空间由 OLDDIR 重定位到 NEWDIR
      --waldir=WALDIR    预写日志目录的位置
  -X, --wal-method=none|fetch|stream
                         按指定的模式包含必需的WAL日志文件
  -z, --gzip             对tar文件进行压缩输出
  -Z, --compress=0-9     按给定的压缩级别对tar文件进行压缩输出

一般选项:
  -c, --checkpoint=fast|spread
                         设置检查点方式(fast或者spread)
  -C, --create-slot      创建复制槽
  -l, --label=LABEL      设置备份标签
  -n, --no-clean         出错后不清理
  -N, --no-sync          不用等待变化安全的写入磁盘
  -P, --progress         显示进度信息
  -S, --slot=SLOTNAME    用于复制的槽名
  -v, --verbose          输出详细的消息
  -V, --version          输出版本信息, 然后退出
      --manifest-checksums=SHA{224,256,384,512}|CRC32C|NONE
                         use algorithm for manifest checksums
      --manifest-force-encode
                         hex encode all file names in manifest
      --no-estimate-size do not estimate backup size in server side
      --no-manifest      suppress generation of backup manifest
      --no-slot          防止创建临时复制槽
      --no-verify-checksums
                         不验证校验和
  -?, --help             显示帮助, 然后退出

联接选项:
  -d, --dbname=CONNSTR   连接串
  -h, --host=HOSTNAME    数据库服务器主机或者是socket目录
  -p, --port=PORT        数据库服务器端口号
  -s, --status-interval=INTERVAL
                         发往服务器的状态包的时间间隔 (以秒计)
  -U, --username=NAME    指定连接所需的数据库用户名
  -w, --no-password      禁用输入密码的提示
  -W, --password         强制提示输入密码 (默认)
```

## 6 复制槽介绍

很多时候在主库产生wal日志的时候，还没有传到从库就被覆盖了，为了保证wal日志不被覆盖，postgres 就启用流复制槽，让没有传到从库的wal保存不被覆盖，新的日志继续产生。

```sql
# 创建
SELECT * FROM pg_create_physical_replication_slot('slave01');

# 查看创建的复制槽
SELECT * FROM pg_replication_slots ;

# 删除复制槽, 我没有创建
SELECT * FROM pg_drop_replication_slot('slave01');
```
