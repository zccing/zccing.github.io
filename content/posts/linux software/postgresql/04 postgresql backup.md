---
title: "备份与恢复PostgreSQL数据库"
subtitle: ""
date: 2022-03-25T20:22:45+08:00
lastmod: 2022-03-25T20:22:45+08:00
draft: true
author: ""
authorLink: ""
description: ""
tags: ['PostgreSQL']
categories: ['linux software']

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

pg_basebackup 是一个广泛使用的PostgreSQL 备份工具，它允许我们进行在线和一致的文件系统级备份。这些备份可用于时间点恢复或设置从属/ 备用。

<!--more-->

http://www.postgres.cn/docs/13/continuous-archiving.html

## 1 开启wal归档

```sh
# 创建备份目录
mkdir -pv /data/pgsql_backup/
mkdir -pv /data/pgsql_backup/{basebackup, wal}
chown -R postgres:postgres /data/pgsql_backup
# 创建压缩备份WAL的脚本
cat > /data/archive_wal.sh<<"EOF"
#!/usr/bin/env bash

EXIT_CODE=0
ARCHIVE_FILE=${1}  # %f
ARCHIVE_DIR=${2}   # %p
BACKUP_PATH=${3}   # /data/pg_wal_backup/wal

if [[ -d "${BACKUP_PATH}" ]]; then
    command -v zstd > /dev/null 2>&1
    if [[ $? -gt 0 ]]; then
        test ! -f "${BACKUP_PATH}/${ARCHIVE_FILE}" && gzip < ${ARCHIVE_DIR} > "${BACKUP_PATH}/${ARCHIVE_FILE}"
        # restore_command => gunzip < ${BACKUP_PATH}/${ARCHIVE_FILE} > $ARCHIVE_DIR
    else
        test ! -f "${BACKUP_PATH}/${ARCHIVE_FILE}" && zstd -f -q -o "${BACKUP_PATH}/${ARCHIVE_FILE}" ${ARCHIVE_DIR}
        # restore_command => zstd -d -q -o $ARCHIVE_DIR ${BACKUP_PATH}/${ARCHIVE_FILE}
    fi
    EXIT_CODE=$?
fi

exit ${EXIT_CODE}
EOF
chmod +x /data/archive_wal.sh
# 修改postgresql的配置，开启热备
vim /data/app/pg_data/postgresql.conf
wal_level = replica  # 需要将日志级别设置为replica或更高级别; 可使用默认值
archive_mode = on    # 打开归档备份
archive_command = '/data/archive_wal.sh %f %p /data/pgsql_backup/wal'
```

## 2 创建基础备份

```sh
# 创建一个备份账号
create user repl  REPLICATION  LOGIN ENCRYPTED  PASSWORD 'password';

# 开启权限
cat >> /etc/postgresql/13/main/pg_hba.conf<<EOF
host    replication     all             169.3.248.0/21          md5
EOF

# 备份
pg_basebackup -U postgres -Ft -Pv -Xs -z -Z5 -h localhost -p 5432 -D /data/pgsql_backup/basebackup/$(date +%F)
```

## 3 WAL恢复

```sh
# 创建恢复脚本
cat > /data/recovery_wal.sh<<"EOF"
#!/usr/bin/env bash

EXIT_CODE=0
ARCHIVE_FILE=${1}  # %f
ARCHIVE_DIR=${2}   # %p
BACKUP_PATH=${3}   # /data/pg_wal_backup/wal

if [[ -d "${BACKUP_PATH}" ]]; then
    command -v zstd > /dev/null 2>&1
    if [[ $? -gt 0 ]]; then
        # test ! -f "${BACKUP_PATH}/${ARCHIVE_FILE}" && gzip < ${ARCHIVE_DIR} > "${BACKUP_PATH}/${ARCHIVE_FILE}"
        gunzip < ${BACKUP_PATH}/${ARCHIVE_FILE} > $ARCHIVE_DIR
    else
        # test ! -f "${BACKUP_PATH}/${ARCHIVE_FILE}" && zstd -f -q -o "${BACKUP_PATH}/${ARCHIVE_FILE}" ${ARCHIVE_DIR}
        zstd -d -q -o $ARCHIVE_DIR ${BACKUP_PATH}/${ARCHIVE_FILE}
    fi
    EXIT_CODE=$?
fi

exit ${EXIT_CODE}
EOF
chmod +x /data/recovery_wal.sh
# 停止数据库进程
systemctl stop postgresql@14-main.service 
# 备份旧的数据库
mv /data/pgsql /data/pgsql_$(date +%F)
# 创建新数据库目录
mkdir /data/pgsql/
# 拷贝最新的基本备份到新的数据库目录
cp /data/pgsql_backup/basebackup/2022-03-25/* /data/pgsql/
# 解压基础备份文件到新的数据库目录
cd /data/pgsql/
tar -zxf base.tar.gz 
tar -zxf pg_wal.tar.gz -C pg_wal/
rm -f base.tar.gz pg_wal.tar.gz 
chown -R postgres:postgres /data/pgsql
# 修改postgresql的恢复配置
vim /etc/postgresql/14/main/postgresql.conf
restore_command = '/data/recovery_wal.sh %f %p /data/pgsql_backup/wal'
# 创建recovery.signal文件，
touch /data/pgsql/recovery.signal
chown -R postgres:postgres /data/pgsql/recovery.signal
chmod 700 /data/pgsql/recovery.signal
# 修改pg_hba.conf限制用户连接，并开启数据库
mv /etc/postgresql/14/main/pg_hba.conf /etc/postgresql/14/main/pg_hba.conf.bak
## 修改pg_hba
cat > /etc/postgresql/14/main/pg_hba.conf<<"EOF"
local   all             postgres                                peer
EOF
chown -R postgres:postgres /etc/postgresql/14/main/pg_hba.conf
## 开启数据库
## 如果中途遇见错误，可以重启数据库，恢复完成会删除recovery.signal文件
systemctl start postgresql@14-main.service 
# 检查数据，如果恢复完成了，修改pg_hba.conf 允许用户连接
mv /etc/postgresql/14/main/pg_hba.conf.bak /etc/postgresql/14/main/pg_hba.conf
systemctl restart postgresql@14-main.service
```

## 4 参考PostgreSQL函数

* `select pg_switch_wal();`: 手动归档一个最早的wal