---
title: "squid学习笔记"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: true
keywords: ['linux代理', 'squid','proxy']
description: "通过二进制方式安装ss，用来做国际优化"
tags: ['linux']
categories: ['linux']
---

介绍编译、二进制、存储库三种方式焊装squid，并配置squid，和支持https代理
<!--more-->


# proxy学习笔记

## 安装squid

> 最简单的方法就是通过存储库 yum 安装
> 也可以源码编译安装，或者二进制安装

### yum安装

```shell
yum updata && yum install -y squid
```

### 编译安装

> 最新源码地址：http://www.squid-cache.org/Versions/

```shell
## 安装依赖包
yum install gcc-c++ make gcc openssl openssl-devel

## 创建启动用户，目录和更改目录权限
useradd squid -U -s /sbin/nologin -d /data/squid/data-cache
mkdir /usr/local/squid /data/squid/
chmod -R squid:squid /data/squid/

## 下载源码并解压到当前目录
wget http://www.squid-cache.org/Versions/v4/squid-4.4.tar.gz
tar -zxf squid-4.4.tar.gz

## 进入解压后的目录，并编译，可以通过 ./configure --help 来查看参数
cd squid-4.4/
./configure \
--prefix=/usr/local/squid \
--datadir=/data/squid/ \
--sysconfdir=/etc/squid \
--with-openssl \
--enable-snmp \
--enable-ssl-crtd \
--with-default-user=squid \
--disable-ipv6 \

## 查看预配置是否通过
echo $?

## 编译安装
make -j4 && make install
```

## 服务端配置

> /usr/sbin/squid：提供squid的主程式啊！
> /var/spool/squid：就是预设的squid缓存放置的目录。
> /usr/lib64/squid/：提供squid额外的控制模组，尤其是影响认证密码方面的程式，都是放在这个目录下的；
> /etc/squid/: squid的配置文件目录
>
> 以上是通过存储库yum安装后的文件目录，如果编译或者二进制安装，目录为自定义的

* selinux
    配置目录(/etc/squid/ 内的文件) 类型是squid_conf_t 的样式， 而缓存目录的类型则是squid_cache_t 的类型，且上层类型(/var/spool/)应该是要成为var_t 之类的才行。修改的方法就是通过chcon 来处理即可。

* 防火墙
    放行配置的服务开放端口,默认为3128
    ```shell
    centos 6
    vim /usr/local/virus/iptables/iptables.allow
    iptables -A INPUT -i $EXTIF -p tcp -s 0.0.0.0/0 --dport 3128 -j ACCEPT
    /usr/local/virus/iptables/iptables.rule

    centos 7
    firewall-cmd --add-port=3128/tcp --permanent --zone=public
    firewall-cmd --reload
    ```

* /etc/squid/mime.conf
    默认就好，设定支持代理的文件格式，就是所谓的mime格式，默认即可满足需求，如果有不支持的，代理不了的mime格式，可以手动更改

* /etc/squid/squid.conf
  * acl语法
    ```shell
    acl <acl名称> <要控制的acl类型> <设定的内容>
    ```
    * 自订的acl名称: 自定义的一个名称
    * acl类型有src（源地址）， dst(目标地址)， srcdomain（来源ddns）， dstdomain(网站主机名)， url_regex（正则表达网址）， urlpath_regex（正则表达文件）
    * 对应类型的IP地址或者域名
    * 实例
        ```conf
        vim /etc/squid/squid.conf
        acl vbirdlan src 192.168.1.0/24 192.168.100.0/24 ## 192.168.1.0/24 192.168.100.0/24 两个网络，all代表全部
        acl vbirdlan2 src 192.168.1.100-192.168.1.200/24 ## 192.168.1.100-192.168.1.200/24这个网段
        acl vbirdksu srcdomain .ksu.edu.tw ## .ksu.edu.tw的域名
        acl ksuurl url_regex ^http://www.ksu.edu.tw/cht/.* ## 依照 http://www.ksu.edu.tw/cht/. 开头的所有网址
        acl sexurl urlpath_regex /sexy.*\.jpg$ ## 出现/sexy名称并以jpg结尾的
        acl dropdomain dstdomain "/etc/squid/dropdomain.txt" ## 读取文件中内容来设定主机名，
        ```
  * http_access和cache语法
    * 语法
        > 有顺序的，从上往下匹配，匹配到后就不往后匹配了
        ```shell
        http_access <acl定义的名称> deny|allow
        cache <acl定义的名称> deny|allow
        ```
    * 实例
        1. 假设我要放行内部网路192.168.1.0/24, 192.168.100.0/24 这两段网域，然后拒绝对外的色情相关图片， 以及facebook.com 网站，那么就应该要这样做：
            ```conf
            vim /etc/squid/squid.conf
            acl vbirdlan src 192.168.1.0/24 192.168.100.0/24
            acl dropdomain dstdomain .facebook.com
            acl dropsex urlpath_regex /sexy.*jpg$
            http_access deny dropdomain   <==这三行的『顺序』很重要！
            http_access deny dropsex
            http_access allow vbirdlan
            ```
            > 注意，如果先放行了vbirdlan才抵挡dropdomain时，你的设定可能会失败！因为内网已经先放行，因此后面的规则不会比对，那么facebook.com就无法被抵挡了！这点得要很注意才行！ 通常的作法是，先将要拒绝的写上去，然后才写要放行的资料就好了。

        2. 只要网址列出现.php 结尾的，就不予以快取！
            ```conf
            vim /etc/squid/squid.conf
            acl denyphp urlpath_regex \.php$
            cache deny denyphp
            ```
  * 磁碟中快取的存在时间
    * 语法：
        ```shell
        refresh_pattern <regex> <最小时间> <百分比> <最大时间>
        ```
        > regex: 使用的是正规表示法来分析网址列的资料
        > 最小时间：单位是分钟，当取得这个资料的时间超过这个设定值，则该资料会被判定为旧资料
        > 百分比：这个项目与『最大时间』有关，当该资料被抓取到快取后，经过最大时间的多少百分比时，该资料就会被重抓。
        > 最大时间：与上一个设定有关，就是这个资料存在快取内的最长时间。如上面第一行，最大时间为10080 分钟，但是当超过此时间的 20% (2016分钟) 时，这个资料也会被判定为旧资料。
    * 实例：
        在网址列出现.vbird. 字样时，该资料为暂时使用的，因此2 小时后就算旧资料。而最长保留在快取给她一天的时间， 且经过50% 的时间后，就被判定为旧资料吧！
        ```conf
        vim /etc/squid/squid.conf
        refresh_pattern ^ftp: 1440 20% 10080
        refresh_pattern ^gopher: 1440 0% 1440
        refresh_pattern -i (/cgi-bin/|\?) 0 0% 0
        refresh_pattern \.vbird\. 120 50% 1440
        refresh_pattern . 0 20% 4320
        ```
  * 全局配置
    ```conf
    ## 1.信任用户与目标控制，透过acl定义出localhost等相关用户
    acl manager proto cache_object  ## 定义manager为管理功能
    acl localhost src 127.0. 0.1/32 ## 定义localhost为本机来源
    acl localhost src ::1/128
    acl to_localhost dst 127.0.0.0/8 0.0.0.0/32 ## 定义to_localhost可连线到本机
    acl to_localhost dst ::1/128

    ## 2.信任用户与目标控制，定义可能使用这部proxy的外部用户(内网)
    acl localnet src 10.0.0.0/8   ## 可发现底下都是private IP的设定
    acl localnet src 172.16.0.0/12
    acl localnet src 192.168.0.0/16
    acl localnet src fc00::/7
    acl localnet src fe80::/10
    ## 上述资料设定两个用户(localhost, localnet) 与一个可取得目标(to_localhost)

    ## 3.定义可取得的资料埠口所在！
    acl SSL_ports port 443 ## https  连线加密的埠口设定
    acl Safe_ports port 80 ## http   公认标准的协定使用埠口
    acl Safe_ports port 21 ## ftp
    acl Safe_ports port 443 ## https
    ## 定义出SSL_ports 及标准的常用埠口Safe_ports 两个名称
    acl CONNECT method CONNECT

    ## 4.定义这些名称是否可放行的标准依据(有顺序喔！)
    http_access allow manager localhost   ## 放行管理本机的功能
    http_access deny manager              ## 其他管理来源都予以拒绝
    http_access deny !Safe_ports          ## 拒绝非正规的埠口连线要求
    http_access deny CONNECT !SSL_ports   ## 拒绝非正规的加密埠口连线要求
    ## 这个位置为你可以写入自己的规则的位置喔！不要写错了！有顺序之分的！
    http_access allow localnet            ## 放行内部网路的用户来源
    http_access allow localhost           ## 放行本机的使用
    http_access deny all                  ## 全部都予以拒绝啦！

    ## 5.网路相关参数，最重要的是那个定义Proxy协定埠口的http_port
    http_port 3128      ## Proxy预设的监听用户端要求的埠口，是可以改的
    ## 其实，如果想让proxy server/client 之间的连线加密，可以改用https_port (923)

    ## 6.缓存与记忆体相关参数的设定值，尤其注意记忆体的计算方式
    hierarchy_stoplist cgi-bin ?    ## hierarchy_stoplist后面的关键字(此例为cgi-bin)
    #若发现在用户端所需要的网址列，则不缓存(避免经常变动的资料库或程式讯息)
    cache_mem 300 MB  ## 给proxy额外的记忆体，用来处理最热门的缓存资料(需自己加)，物理最少要是缓存容量G*10 + 15M + 此处定义值的两倍，比如缓存10G * 10 + 15M + 300M = 415M, 所需物理内存最少要是1G

    ## 7.磁碟缓存，亦即放置缓存资料的目录所在与相关设定
    cache_dir ufs /var/spool/squid 10000 16 256  ## 100代表缓存的可用硬盘的容量，16代表第一层次目录共有16个，256 代表每层次目录内部再分为256 个次目录，后两个数字默认有16 256 以及64 64 这两种配置（官方推荐最佳配置）
    coredump_dir /var/spool/squid
    #底下的四个参数得要自己加上来喔！旧版才有这样的预设值！
    minimum_object_size 1024 KB    ## 小于多少KB的资料不要放缓存，0为不限制
    maximum_object_size 102400 KB ## 与上头相反，大于4 MB的资料就不缓存到磁碟
    cache_swap_low 90    ## 与下一行有关，减低到剩下90%的磁碟缓存为止
    cache_swap_high 95   ## 当磁碟使用量超过95%就开始删除磁碟中的旧缓存

    ## 8.其他可能会用到的预设值！参考参考即可，并不会出现在设定档中。
    access_log /var/log/squid/access.log squid ## 曾经使用过squid的用户记录
    ftp_user Squid@   ## 当以Proxy进行FTP代理匿名登入时，使用的帐号名称
    ftp_passive on    ## 若有代理FTP服务，使用被动式连线
    refresh_pattern ^ftp: 1440 20% 10080
    refresh_pattern ^gopher: 1440 0% 1440
    refresh_pattern -i (/cgi-bin/|\?) 0 0% 0
    refresh_pattern . 0 20% 4320
    #上面这四行与缓存的存在时间有关，底下内文会予以说明
    cache_mgr root               ## 预设的proxy管理员的email
    cache_effective_user squid   ## 启动squid PID的拥有者
    cache_effective_group squid  ## 启动squid PID的群组
    ## visible_hostname  ## 有时由于DNS的问题，找不到主机名会出错，就得加上此设定
    ipcache_size 1024   ## 以下三个为指定IP进行缓存的设定值
    ipcache_low 90
    ipcache_high 95
    ```

## 使用https连接代理服务器

> 防止GFW重定向连接
> 使用certbot-auto 生成证书

## pac设置

>