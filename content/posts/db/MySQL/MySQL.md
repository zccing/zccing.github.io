---
title: "安装mysql"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-07T18:54:18+08:00
draft: false
keywords: ['mysql', 'mysql安装','二进制安装mysql','编译安装mysql']
description: "linux安装最新版的mysql记录，通过存储库、二进制、编译三种方式进行安装mysql"
tags: ['mysql']
categories: ['数据库']
---

linux安装最新版的mysql记录，通过存储库、二进制、编译三种方式进行安装mysql
<!--more-->


# 安装mysql

## 通过yum软件库安装Mysql

> 以下命令对于启用了dnf的系统，用dnf替换命令中的 yum
>
> 发布软件包安装在系统上，yum update 命令（或dnf-enabled系统的dnf升级）进行的任何系统范围的更新都会自动升级系统上的MySQL软件包，并且还会替换任何本地的第三方软件包在MySQL Yum存储库中找到替代者
>
> 默认配置文件路径：</br>
配置文件：/etc/my.cnf </br>
日志文件：/var/log# var/log/mysqld.log </br>
服务启动脚本：/usr/lib/systemd/system/mysqld.service </br>
socket文件：/var/run/mysqld/mysqld.pid</br>

### 一、安装MySQL RPM

Yum存储库添加到系统的存储库列表中

* 到MySQL存储库下载适用linux版本的发行包，[点我下载](http://dev.mysql.com/downloads/repo/yum/.)</br>
* 使用以下命令安装下载的发行包</br>

    ```mysql
    rpm -Uvh 发行包名称
    ```

### 二、选择一个mysql安装版本

> 当使用MySQL Yum存储库时，默认选择安装MySQL的最新GA版本进行安装。如果这是你想要的，你可以跳到下一步， 用Yum安装MySQL

* 查看MySQL Yum存储库中的所有子存储库，并查看其中哪些被启用或禁用

```bash
yum repolist all | grep mysql
# 查看MySQL Yum存储库中的所有子存储库，并查看其中哪些被启用或禁用
sudo yum-config-manager --disable mysql**-community
# 禁用最新的GA系列的子库
sudo yum-config-manager --enable mysql**-community
# 启用特定GA系列的子库
```

* 如若不支持yum-config-manager命令，可以通过修改etc/yum.repos.d/mysql-community.repo来解决，

```bash
vim etc/yum.repos.d/mysql-community.repo
# 打开文件,找到要配置的子库的条目，然后编辑该enabled选项,指定 enabled=0禁用子库，或 enabled=1启用子库
yum repolist enabled | grep mysql
# 验证是否已启用和禁用了正确的子库
```

> 为一个发行版系列启用子库。当启用多个版本系列的子库时，Yum将使用最新的系列

### 三、安装mysql

```bash
yum install mysql-community-server
# 安装MySQL，装MySQL服务器的软件包以及其他必需的软件包
```

### 四、启动MySQL服务器

```bash
service mysqld start
# 启动MySQL服务
systemctl start mysqld.service
# EL7的启动命令，首选
service mysqld status
# 查看MySQL服务的状态
```

### 五、开机启动mysql

```bash
systemctl enable mysqld
systemctl daemon-reload
```

### 六、MySQL服务器初始化（仅适用于MySQL 5.7)

* mysql安装完成之后，在/var/log/mysqld.log文件中给root生成了一个默认密码（初始化）。通过下面的方式找到root默认密码，然后登录mysql进行修改
    ```sql
    grep 'temporary password' /var/log/mysqld.log
    # 显示初始的超级用户的密码
    mysql_secure_installation
    # 初始mysql，后边可以不用操作
    mysql -uroot -p
    # 使用初始密码登陆
    mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
    mysql> set password for 'root'@'localhost'=password('MyNewPass4!');
    # 更改密码，两条命令选择一条即可
    ```
* mysql5.7默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包含：大小写字母、数字和特殊符号，并且长度不能少于8位。否则会提示ERROR 1819
    ```sql
    show variables like '%password%';
    # 通过msyql环境变量可以查看密码策略的相关信息
    >  validate_password_policy:
    #  密码策略，默认为MEDIUM策略
    validate_password_dictionary_file：
    # 密码策略文件，策略为STRONG才需要
    validate_password_length：
    # 密码最少长度
    validate_password_mixed_case_count：
    # 大小写字符长度，至少1个
    validate_password_number_count:
    # 数字至少1个
    validate_password_special_char_count：
    # 特殊字符至少1个
    ```

    共有以下几种密码策略：

    | 策略        | 检查规则                                                                      |
    | ----------- | ----------------------------------------------------------------------------- |
    | 0 or LOW    | Length                                                                        |
    | 1 or MEDIUM | Length; numeric, lowercase/uppercase, and special characters                  |
    | or STRONG   | Length; numeric, lowercase/uppercase, and special characters; dictionary file |
    MySQL官网密码策略详细说明：[点击查看](http://dev.mysql.com/doc/refman/5.7/en/validate-password-options-variables.html#sysvar_validate_password_policy)</br>
    </br>
    修改密码策略
    ```vim
    vim /etc/my.cnf
    # 打开mysql的配置文件
    validate_password_policy=0
    # 添加一种密码策略，选择0（LOW），1（MEDIUM），2（STRONG）其中一种，选择2需要提供密码字典文件
    validate_password = off
    # 如果不需要密码策略，添加my.cnf文件中添加这一行的配置禁用即可
    systemctl restart mysqld
    # 重新启动mysql服务使配置生效：
    ```

## 从源代码编译安装

### 配置编译环境

* yum安装需要的库文件
    ```bash
    yum install wget gcc gcc-c++ make perl ncurses ncurses-devel openssl openssl-devel libaio libaio-devel bison-devel bison
    ```

* 安装cmake
    ```bash
    yum install gcc-c++  # 安装c++支持
    wget https:# cmake.org/files/v3.11/cmake-3.11.4-Linux-x86_64.tar.gz  # 官网下载最新的cmake源文件压缩包
    tar -zxf cmake-3.11.4-Linux-x86_64.tar.gz   # 解压源文件到当前目录
    cd cmake-3.11.4-Linux-x86_64  # 进入cmake源文件目录
    ./bootstrap --prefix=/opt/cmake/  # 编译安装到/opt/cmake目录下
    make -j 16  # 16线程编译，可以加快速度
    make install
    ln -s /opt/cmake/bin/* /usr/local/bin/  # 软连接到系统环境变量目录
    cmake --version  # 查看当前cmake版本
    ```

* 安装boost(可选，因为我安装的源代码是包含boost头的，建议不安装和编译，节省时间)
    ```bash
    cd ~/src
    wget https:# dl.bintray.com/boostorg/release/1.68.0/source/boost_1_68_0.tar.gz   # 下载
    tar -zxf boost_1_68_0.tar.gz  # 解压缩
    cd boost_1_68_0

    # 配置编译选项
    ./bootstrap.sh \
    --exec-prefix=/opt/boost/ \   # 安装目录
    --with-libraries=all \        # 需要编译的boost库，all表示所有
    --with-toolset=gcc \          # 指定编译时使用哪种编译器
    --with-icu \                  # 开启正则表达式支持icu
    --with-python=python          # 指定Python可执行文件

    # 编译安装
    ./b2 install toolset=gcc variant=release threading=multi toolset=gcc --prefix=/opt/boost

    # 添加环境变量
    mkdir /opt/lib
    echo "export C_INCLUDE_PATH=/opt/lib:$C_INCLUDE_PATH" > /etc/profile
    echo "export CPLUS_INCLUDE_PATH=/opt/lib:$CPLUS_INCLUDE_PATH" > /etc/profile
    echo "export LD_LIBRARY_PATH=/opt/lib:$LD_LIBRARY_PATH" > /etc/profile
    echo "export LIBRARY_PATH=/opt/lib:$LIBRARY_PATH" > /etc/profile
    source /etc/profile
    ln -s /opt/boost/lib/* /opt/lib/
    ldconfig -v
    ```

* 添加用户和创建安装、数据库文件目录
    ```bash
    groupadd dba
    useradd -r -g dba -s /bin/false dba
    mkdir /opt/mysql
    mkdir -p /data/mysql
    chown -R dba:dba /data/mysql
    ```

### 编译mysql

* 下载mysql源包
    ```bash
    wget http://mirrors.163.com/mysql/Downloads/MySQL-8.0/mysql-boost-8.0.12.tar.gz   # 从网易镜像站下载源文件
    wget http://mirrors.163.com/mysql/Downloads/MySQL-8.0/mysql-boost-8.0.12.tar.gz.md5   # 下载校验md5
    md5sum -c mysql-boost-8.0.12.tar.gz.md5   # 校验源文件
    rm -f mysql-boost-8.0.12.tar.gz.md5
    tar -zxf mysql-boost-8.0.12.tar.gz  # 解压
    ```

* 编译安装和初始化mysql
    ```bash
    # 编译
    mkdir bld
    cd bld
    cmake .. \
    -DCMAKE_INSTALL_PREFIX=/opt/mysql \
    -DENABLE_DOWNLOADS=1 \
    -DMYSQL_DATADIR=/data/mysql \
    -DSYSCONFDIR=/etc \
    -DMYSQL_TCP_PORT=3306 \
    -DWITH_INNOBASE_STORAGE_ENGINE=1 \
    -DWITH_PARTITION_STORAGE_ENGINE=1 \
    -DWITH_ARCHIVE_STORAGE_ENGINE=1 \
    -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
    -DSYSTEMD_PID_DIR=/run \
    -DWITH_SYSTEMD=ON \
    -DWITH_BOOST=/root/src/mysql-8.0.12/boost \
    -DWITH_INNODB_MEMCACHED=ON \
    -DWITH_GMOCK=/root/src/mysql-8.0.12/GoogleTest
    make -j 4
    make install

    # 初始化
    cd /opt/mysql
    mkdir mysql-files
    chown dba:dba mysql-files
    chmod 750 mysql-files
    bin/mysqld --initialize --user=dba --datadir=/data/mysql/ # 记录随机的密码
    bin/mysql_ssl_rsa_setup  # 开启openssl，并生成key
    cp usr/lib/systemd/system/mysqld.service /usr/lib/systemd/system/ # 添加到系统服务
    vim /usr/lib/systemd/system/  # 把这两个更改下，User=dba Group=dba
    systemctl daemon-reload
    systemctl start mysql
    mysql -uroot -p # 输入之前获得的随机码
    ALTER USER 'root'@'localhost' IDENTIFIED WITH sha256_password BY 'MyNewPass4!'; # 修改root密码，并更改加密方式
    ```

## 二进制安装mysql

和源码安装一样只是省去了编译的过程

## 修改配置文件

* 修改/etc/my.cnf配置文件，如下所示：
    ```ini
    [mysqld]
     # 开启慢查询日志
    slow_query_log = ON
    slow_query_log_file = /var/log/mysql/slow.log
    long_query_time = 10
    log_queries_not_using_indexes = 1

    # 开启操作日志
    general_log=ON
    general_log_file=/var/log/mysql/operation.log

    # 错误日志
    log_error=/var/log/mysql/error.log

    # 更改默认编码
    character_set_server=utf8mb4
    init_connect='SET NAMES utf8mb4'

    # 开启openssl，在初始化的时候执行bin/mysql_ssl_rsa_setup 命令，可以在数据目录里生成key
    ssl_ca=/data/mysql/ca.pem               # ca
    ssl_capath=/data/mysql                  # ca证书路径
    ssl_cert=/data/mysql/server-cert.pem    # 服务器公钥
    ssl-key=/data/mysql/server-key.pem      # 服务器私钥
    # ssl_cipher=                           # 加密方法
    # ssl_crl=
    # ssl_crlpath=
    # ssl_fips_mode=OFF
    # ssl_key=server-key.pem

    [client]
    # 本地启用ssl登录
    ssl-ca=ca.pem
    ssl_capath=/data/mysql
    ssl-cert=client-cert.pem
    ssl-key=client-key.pem

    # socket连接文件的位置
    socket=/tmp/mysql.sock
    ```
* openssl创建证书
    如果你不准备用mysql自动生成的证书，可以手动重新生成你需要的
    ```bash
    ##########################
    #                        ＃
    #      生成ＣＡ   　　　　　＃
    #                        ＃
    ##########################
    # 生成一个ca私钥
    openssl genrsa 2048 > ca-key.pem
    # 根据 CA 私钥生成一个新的数字证书，根据提示回答相应问题，“..”表示留空，openssl req --help 查看各变量的意思
    openssl req -sha1 -new -x509 -nodes -days 3650 -key ca-key.pem > ca-cert.pem

    ##########################
    #                     　 ＃
    # 服务端的RSA私钥和数字证书 ＃
    #                     　 ＃
    ##########################
    # 创建服务器端的私钥和一个证书请求文件，记住最后输入的密码
    openssl req -sha1 -newkey rsa:2048 -days 3650 -nodes -keyout server-key.pem > server-req.pem
    # 将生成的私钥转换为 RSA 私钥文件格式
    openssl rsa -in server-key.pem -out server-key.pem
    # 使用原先生成的 CA 证书来生成一个服务器端的数字证书
    openssl x509 -sha1 -req -in server-req.pem -days 3650 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 > server-cert.pem

    ##########################
    #                     　 ＃
    # 客户端的RSA私钥和数字证书 ＃
    #                     　 ＃
    ##########################
    # 生成一个私钥和证书
    openssl req -sha1 -newkey rsa:2048 -days 3650 -nodes -keyout client-key.pem > client-req.pem
    # 私钥转换为 RSA 私钥文件格式
    openssl rsa -in client-key.pem -out client-key.pem
    # 使用原先生成的 CA 证书来生成一个客户端的数字证书
    openssl x509 -sha1 -req -in client-req.pem -days 3650 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 > client-cert.pem
    ```
* openssl创建的文件
  
    | 文件 | 用途 |
    |-------------|-----------|
    | ca-cert.pem | CA 证书, 用于生成服务器端/客户端的数字证书. |
    | ca-key.pem | CA 私钥, 用于生成服务器端/客户端的数字证书. |
    | server-key.pem | 服务器端的 RSA 私钥 |
    | server-req.pem | 服务器端的证书请求文件, 用于生成服务器端的数字证书. |
    | server-cert.pem | 服务器端的数字证书. |
    | client-key.pem | 客户端的 RSA 私钥 |
    | client-req.pem | 客户端的证书请求文件, 用于生成客户端的数字证书. |
    | client-cert.pem | 客户端的数字证书. |

## mysql的命令

* 重新启动mysql服务，查看数据库默认
    ```sql
    systemctl restart mysqld
    mysql -uroot -p
    # 登陆sql
    show variables like 'char%';
    # 查看默认数据库编码
    ```
* 更改之前创建的数据库的编码
    ```sql
    # db_name改成数据库名称
    show databases;
    # 查看数据库表名称
    alter database db_name CHARACTER SET utf8;
    # db_name改成数据库名称
    ```
* 把在所有数据库的所有表的所有权限赋值给位于所有IP地址的root用户。

    ```sql
    mysql> grant all privileges on *.* to root@'%'identified WITH sha256_password by 'password';
    ```

* 如果是新用户而不是root，则要先新建用户

    ```sql
    mysql>create user 'username'@'%' identified WITH sha256_password by 'password';
    ```
