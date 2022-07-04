---
title: "PostgreSQL中间件Pgpool-II安装配置"
date: 2022-03-05T19:52:27+08:00
draft: true
keywords: []
description: ""
tags: ['PostgreSQL']
categories: ['DB']
author: "Cc"
autoCollapseToc: false
---

利用pgpool-II对PostgreSQL13集群做读写分离, 此文章的前提是已经安装好了两台postgresql数据库服务器并配置好了主从模式

<!--more-->



## 介绍

**两台数据库服务器；主数据库服务器IP:169.3.250.215，备数据库服务器IP:169.3.254.215, 我将pgpool分别安装到了两台数据库服务器上**

## 安装

```bash
apt update && apt install postgresql-13-pgpool2 pgpool2
```

## 账号和权限

* 访问权限

    配置方式和postgresql的hba一样, 文件位置`/etc/pgpool2/pool_hba.conf`

    用于对访问 PGPool-II 中间件的请求实施访问认证控制。同时，因为 PGPool-II 从架构上位于 PostgreSQL 之前，因此请求需要先通过 PGPool-II 的认证控制，随后通过 PostgreSQL 的认证控制

    ```bash
    cat >> /etc/pgpool2/pool_hba.conf<<EOF
    host    all         all         169.3.250.215/21      md5
    EOF
    ```

* pgpool-ii的管理账号

    用来管理pgpool-II集群的账号设置, 文件位置`/etc/pgpool2/pcp.conf`

    ```bash
    echo pgpool:$(pg_md5 -u pgpool <password>) >>/etc/pgpool2/pcp.conf
    ```

*  PGPool-II对访问数据库的账号认证

    由于pgpoll-II无法获取PostgreSQL数据库上的用户密码信息，因此其通过检查pool_passwd内用户名及密码的方式，校验请求输入的用户名及密码是否正确。另外，当请求通过pgpoll-II认证后，pgpoll-II 将使用 pool_passwd 内保存的用户名及密码，连接后端 PostgreSQL 数据库。文件位置`/etc/pgpool2/pool_passwd`

    ```bash
    # 将postgresql当前所有账号和密码(密文)添加到pool_passwd文件中
    psql -U postgres -h localhost -t -c "select rolname || ':' || rolpassword from pg_authid where rolpassword is not null ;" > /etc/pgpool2/pool_passwd
    ```

* 免密管理pgpool-ii

    ```bash
    echo "169.3.250.215:port:pgpool:<password>" >>/etc/pgpool2/.pcppass
    echo "169.3.254.215:port:pgpool:<password>" >>/etc/pgpool2/.pcppass
    ```

## 配置

pgpool的配置文件目录: `/etc/pgpool2/pgpool.conf`

```bash
vim /etc/pgpool2/pgpool.conf



# - pgpool Connection Settings -
listen_addresses = '*'
port = '9999'


# - Backend Connection Settings -
backend_hostname0 = '169.3.250.215'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/data/pgsql'
# 控制后台程序的行为，有三种值：
# ALLOW_TO_FAILOVER: 允许故障转移或者从后台程序断开。本值为默认值。
#   指定本值后，不能同时指定 DISALLOW_TO_FAILOVER 。
# DISALLOW_TO_FAILOVER: 不允许故障转移或从后台程序断开。
#   当你使用Heartbeat或Pacemaker等HA（高可用性）软件保护后端时，此功能很有用。
#   你不能同时使用ALLOW_TO_FAILOVER指定。
# ALWAYS_MASTER: 这仅在流复制模式下有用。
#   如果将此标志设置为后端之一，则Pgpool-II将不会通过检查后端找到主节点。
#   而是始终将设置了标志的节点视为主要节点。
backend_flag0 = 'DISALLOW_TO_FAILOVER' # 不开启故障转移
backend_application_name0 = 'master'


backend_hostname1 = '169.3.254.215'
backend_port1 = 5432
backend_weight1 = 2
backend_data_directory1 = '/data/pgsql'
backend_flag1 = 'DISALLOW_TO_FAILOVER'
backend_application_name1 = 'slave01'

# - Authentication -
# 使用 pool_hba.conf 来进行客户端认证，on表示同意
enable_pool_hba = on


# - Where to log -
log_destination = 'syslog'
log_connections = on

#------------------------------------------------------------------------------
# STREAMING REPLICATION MODE
#------------------------------------------------------------------------------

# - Streaming -
# 基于流复制的延迟检查的间隔，单位为秒
sr_check_period = 10
# 执行流复制检查的用户。用户必须存在于所有的PostgreSQL后端上。
sr_check_user = 'nobody'
# 用于执行检查的用户的密码, 留空将从`/etc/pgpool2/pool_passwd`中查找密码
sr_check_password = ''
# 流复制检查的数据库
sr_check_database = 'postgres'

delay_threshold = 10000000


#------------------------------------------------------------------------------
# HEALTH CHECK GLOBAL PARAMETERS
#------------------------------------------------------------------------------

# 定期尝试连接到后台以检测服务器是否在服务器或网络上有问题。 这种错误检测过程被称为“健康检查”。如果检测到错误， 则 pgpool-II 会尝试进行故障恢复或者退化操作。本参数指出健康检查的间隔，单位为秒
health_check_period = 10
# 本参数用于避免健康检查在例如网线断开等情况下等待很长时间。 超时值的单位为秒。默认值为 20, 0 禁用超时（一直等待到 TCP/IP 超时）。
health_check_timeout = 20
health_check_user = 'nobody'
# 用于执行健康检查的用户的密码, 留空将从`/etc/pgpool2/pool_passwd`中查找密码
health_check_password = ''
# 执行健康检查的数据库名
health_check_database = 'postgres'
# 检查失败重试的最大次数。
health_check_max_retries = 3
# 重试等待之前等待的时间, 单位:秒
health_check_retry_delay = 1
# 使用 connect() 系统调用时候放弃连接到后端的超时毫秒值。 默认为 10000 毫秒。
connect_timeout = 10000


#------------------------------------------------------------------------------
# LOAD BALANCING MODE
#------------------------------------------------------------------------------
# 当设置为 on时，SELECT 查询将被分发到每个后台程序上用于负载均衡。
load_balance_mode = on
```