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
categories: ['DB']

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

## wal

```sh
mkdir -p /data/pgsql_archive/wal/
cat > /data/pgsql_archivearchive_wal.sh<<EOF
if [[ -d "${BACKUP_PATH}" ]]; then
    command -v zstd > /dev/null 2>&1
    if [[ $? -gt 0 ]]; then
        gzip < ${ARCHIVE_DIR} > "${BACKUP_PATH}/${ARCHIVE_FILE}"
        # restore_command => gunzip < ${BACKUP_PATH}/${ARCHIVE_FILE} > $ARCHIVE_DIR
    else
        zstd -f -q -o "${BACKUP_PATH}/${ARCHIVE_FILE}" ${ARCHIVE_DIR}
        # restore_command => zstd -d -q -o $ARCHIVE_DIR ${BACKUP_PATH}/${ARCHIVE_FILE}
    fi
    EXIT_CODE=$?
fi

exit ${EXIT_CODE}
EOF
chmod +x /data/pgsql_archivearchive_wal.sh
chown -R postgres:postgres /data/pgsql_archive
```

## 基础备份

```sh
create user repl  REPLICATION  LOGIN ENCRYPTED  PASSWORD 'password';


cat >> /etc/postgresql/13/main/pg_hba.conf<<EOF
host    replication     all             169.3.248.0/21          md5
EOF

pg_basebackup -U repl -Ft -Pv -Xs -z -Z5 -h localhost -p 5432 -D /data/pgsql_archive/$(date +%F)
```

## 全量恢复

```sh
pg_ctl stop

mv /data/pgsql /data/pgsql_$(date +%F)

mkdir /data/pgsql/

cp /data/pgsql_archive/2022-03-25/* /data/pgsql/

cd /data/pgsql/

tar -zxf base.tar.gz 
tar -zxf pg_wal.tar.gz -C pg_wal/

rm -f base.tar.gz pg_wal.tar.gz 

chown -R postgres:postgres /data/pgsql

pg_ctl start

```

## 增量恢复(wal)

