---
title: "openvpn安装配置记录"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['openvpn', '编译安装openvpn','openvpn配置']
description: "编译安装openvpn，并配置lan to lan"
tags: ['VPN']
categories: ['VPN']
---

openvpn的安装配置记录
<!--more-->

# openvpn安装配置记录

## 编译安装openvpn

### 编译安装

* 安装编译依赖库
    ```sh
    yum -y install gcc gcc-c++ make autoconf openssl-devel lzo-devel pam-devel systemd-devel systemd-libs
    ```

* 下载并配置编译环境
    ```sh
    cd /usr/local/src/

    # 需要翻墙，我是翻墙下载过来上传上去的
    wget https://swupdate.openvpn.org/community/releases/openvpn-2.4.6.tar.gz

    # 解压并进入目录
    tar -zxf openvpn-2.4.6.tar.gz
    cd openvpn-2.4.6

    # 配置编译选项，--prefix控制安装目录
    ./configure \
    --prefix=/opt/openvpn \
    --enable-selinux \
    --enable-systemd \
    --enable-server \
    --enable-plugins \
    --enable-management \
    --enable-multihome \
    --enable-port-share \
    --disable-debug \
    --enable-iproute2 \
    --enable-plugin-auth-pam \
    --enable-pam-dlopen \
    --enable-async-push
    ```

* 安装openvpn到/opt/openvpn目录中
    ```sh
    make -j4 && make install
    ```

### 添加环境变量和服务

* 配置环境变量
    ```sh
    vim /etc/profile   # 编辑profile
    export PATH=$PATH:/opt/openvpn/sbin/  # 在最后一行添加，然后就可以直接使用 openvpn命令了
    ```

* 添加man手册
    这步可有可无，无伤大雅
    ```sh
    vim /etc/man_db.conf  # 编辑man配置文件

    # 在MANDATORY_MANPATH文件位置添加以下字符串
    MANDATORY_MANPATH                       /opt/openvpn/share/man
    ```

* 加载内核模块
    ```sh
    lsmod | grep tun    # 查看模块是否有tunnel4和tun模块

    # 安装缺少的模块
    modprobe tunnel4
    modprobe tun

* 添加到系统服务并创建配置文件
    ```sh
    cd /opt/openvpn  # 进入openvpn的安装目录目录
    cp ./lib/systemd/system/openvpn-server@.service /usr/lib/systemd/system/openvpn@server.service  # 拷贝服务文件到系统服务目录
    mkdir -p /etc/openvpn/server/ # 创建配置文件目录
    cd /usr/local/src/openvpn-2.4.6/ # 进入源文件目录
    cp sample/sample-config-files/server.conf /etc/openvpn/server/server.conf # 拷贝配置文件
    ```

## 生成服务端和客户端证书

> 可以创建无数个client的key,每个客户端key的名字不可以一样，创建的文件介绍参照以下表格,迪菲·赫尔曼交换密钥根据KEY-SIZE定义的

| key名字        | 用途介绍            | 安全程度                 |
| -------------- | ------------------- | ------------------------ |
| ca.crt         | ca证书              | 低，客户端和服务端都需要 |
| ca.key         | ca秘钥              | 最高，只有服务端需要     |
| dh.pem         | 迪菲·赫尔曼交换密钥 | 中，只有服务端需要       |
| server.crt     | 服务端证书          | 高，只有服务端需要       |
| server.key     | 服务端秘钥          | 高，只有服务端需要       |
| ta.key（可选） | TLS-auth密钥        | 高，客户端和服务端都需要 |
| client1.crt    | 客户端证书          | 中，只有客户端需要       |
| client1.key    | 客户端秘钥          | 高，只有客户端需要       |
| .........      | N多客户端证书和秘钥 | 中，只有客户端需要       |

### 下载和配置easyrsa

* 下载easyrsa
    ```sh
    cd /usr/local/src/
    git clone https://github.com/OpenVPN/easy-rsa.git
    cp -R easy-rsa /opt/openvpn/
    cd /opt/openvpn/easy-rsa
    cp vars.example vars
    ```
* 配置vars文件
    ```sh
    vim vars

    # 修改一下位置的对应值
    set_var EASYRSA_REQ_COUNTRY     "CN"    # 所在国家
    set_var EASYRSA_REQ_PROVINCE    "Shanghai"  # 所在省
    set_var EASYRSA_REQ_CITY        "Shanghai"  # 所在市
    set_var EASYRSA_REQ_ORG "zhangcc"   # 版权信息
    set_var EASYRSA_REQ_EMAIL       "admin@126.com"    # 邮箱地址
    set_var EASYRSA_REQ_OU          "my openvpn" # 组织或公司名称

    # 可选配置项目
    set_var EASYRSA_PKI            "$PWD/pki"   # 生成的key存放目录，默认当前目录的pki文件夹下
    set_var EASYRSA_OPENSSL        "openssl"    # openssl程序的目录，如果没有创建openssl的环境变量，需要指定绝对路径
    set_var EASYRSA_KEY_SIZE       2048     # 秘钥加密位数，最高4096,只有加密方式为rsa时生效，默认2048
    set_var EASYRSA_ALGO           rsa      # 加密方式,默认rsa，支持rsa和ecc
    set_var EASYRSA_CA_EXPIRE      3650     # 根ca的有效期（单位：天）
    set_var EASYRSA_CERT_EXPIRE     3650    # 证书的有效期（单位：天）
    set_var EASYRSA_SSL_CONF       "$EASYRSA/openssl-easyrsa.cnf"   # openssl的配置文件目录
    ```
* 初始化
    ```bash
    ./easyrsa init-pki  # 初始化
    ```

### 根ca证书

* 生成CA证书和秘钥，秘钥文件在./key/private/下
    ```bash
    ./easyrsa build-ca  # 执行生成CA命令

    # 以下为命令回显
    Note: using Easy-RSA configuration from: ./vars
    Generating a 2048 bit RSA private key
    ...............................................................+++
    .............................................................................................................+++
    writing new private key to '/opt/openvpn/easy-rsa/key/private/ca.key.kxuwxGnM5I'
    Enter PEM pass phrase:  # 输入密码，用来证书签名
    Verifying - Enter PEM pass phrase: # 确认密码
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Common Name (eg: your user, host, or server name) [Easy-RSA CA]:myname  # 输入一个common name

    CA creation complete and you may now import and sign cert requests.
    Your new CA certificate file for publishing is at:
    /opt/openvpn/easy-rsa/key/ca.crt
    ```

### 服务端证书

* 生成服务端证书和秘钥，证书和秘钥文件分别在./key/reqs/和./key/private/下
    ```bash
    ./easyrsa gen-req server nopass   # 生成命令，server为服务端秘钥名称，nopass为不加密秘钥

    # 以下为命令回显
    Note: using Easy-RSA configuration from: ./vars
    Generating a 2048 bit RSA private key
    ...........+++
    ..............+++
    writing new private key to '/opt/openvpn/easy-rsa/key/private/server.key.eF1Ve9ejPO'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Common Name (eg: your user, host, or server name) [server]:MyServerKey # 输入一个common name,不要同于CA的common name

    Keypair and certificate request completed. Your files are:
    req: /opt/openvpn/easy-rsa/key/reqs/server.req
    key: /opt/openvpn/easy-rsa/key/private/server.key
    ```
* 签约服务器端证书
    ```bash
    ./easyrsa sign server server    # 签约server证书为server

    # 以下为命令回显
    Note: using Easy-RSA configuration from: ./vars

    You are about to sign the following certificate.
    Please check over the details shown below for accuracy. Note that this request
    has not been cryptographically verified. Please be sure it came from a trusted
    source or that you have verified the request checksum with the sender.

    Request subject, to be signed as a server certificate for 3650 days:

    subject=
        commonName                = MyServerKey

    Type the word 'yes' to continue, or any other input to abort.
    Confirm request details: yes        # 输入yes

    Using configuration from ./openssl-easyrsa.cnf
    Enter pass phrase for /opt/openvpn/easy-rsa/key/private/ca.key:     # 输入CA的数字签名密码
    Check that the request matches the signature
    Signature ok
    The Subject's Distinguished Name is as follows
    commonName            :ASN.1 12:'MyServerKey'
    Certificate is to be certified until Sep 11 17:11:55 2028 GMT (3650 days)

    Write out database with 1 new entries
    Data Base Updated

    Certificate created at: /opt/openvpn/easy-rsa/key/issued/server.crt

    ```
* 创建Diffie-Hellman，确保key穿越不安全网络
    ```bash
    ./easyrsa gen-dh  # 生成Diffie-Hellman

    # 以下为命令回显
    Note: using Easy-RSA configuration from: ./vars
    Generating DH parameters, 2048 bit long safe prime, generator 2
    This is going to take a long time
    ...............................................................
    ...............................................................
    +..............................................................
    ...............................................................
    +..........+.........................++*++*

    DH parameters of size 2048 created at /opt/openvpn/easy-rsa/key/dh.pem
    ```

### 客户端证书

* 生成客户端证书和秘钥，证书和秘钥文件分别在./key/reqs/和./key/private/下
    ```bash
    ./easyrsa gen-req mysql nopass  # mysql为客户端key的名称，nopass为不加密秘钥

    # 以下为命令回显
    Note: using Easy-RSA configuration from: ./vars
    Generating a 2048 bit RSA private key
    ..........................................................................+++
    .................................................................+++
    writing new private key to '/opt/openvpn/easy-rsa/key/private/mysql.key.q5Kb4bC4el'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Common Name (eg: your user, host, or server name) [mysql]:mysql-to-core     # # 输入一个common name,不要同于CA的common name

    Keypair and certificate request completed. Your files are:
    req: /opt/openvpn/easy-rsa/key/reqs/mysql.req
    key: /opt/openvpn/easy-rsa/key/private/mysql.key
    ```
* 签约客户端秘钥
    ```bash
    ./easyrsa sign client mysql  # 签约客户端mysql秘钥

    # 以下为命令回显
    Note: using Easy-RSA configuration from: ./vars

    You are about to sign the following certificate.
    Please check over the details shown below for accuracy. Note that this request
    has not been cryptographically verified. Please be sure it came from a trusted
    source or that you have verified the request checksum with the sender.

    Request subject, to be signed as a client certificate for 3650 days:

    subject=
        commonName                = mysql-to-core

    Type the word 'yes' to continue, or any other input to abort.
    Confirm request details: yes    # 输入yes同意
    Using configuration from ./openssl-easyrsa.cnf
    Enter pass phrase for /opt/openvpn/easy-rsa/key/private/ca.key:     # 输入CA的数字签名密码
    Check that the request matches the signature
    Signature ok
    The Subject's Distinguished Name is as follows
    commonName            :ASN.1 12:'mysql-to-core'
    Certificate is to be certified until Sep 11 17:42:07 2028 GMT (3650 days)

    Write out database with 1 new entries
    Data Base Updated

    Certificate created at: /opt/openvpn/easy-rsa/key/issued/mysql.crt
    ```
* 生成TLS-auth秘钥（可选）
    > 这一步骤是可选操作。OpenVPN提供了TLS-auth功能，可以用来抵御Dos、UDP端口淹没攻击。出于安全考虑，你可以启用该功能；生成key命令如下：
    ```bash
    openvpn --genkey --secret /opt/openvpn/easy-rsa/key/ta.key
    ```

### 查看和撤销一个证书

* 查看证书的签约
    ```bash
    ./easyrsa show-cert mykey full # mykey代表自己的以.crt为后缀签约过的证书名字
    ./easyrsa show-cert mykey full # mykey代表自己的以.req为后缀的证书名字
    ```
* 撤销证书的签约
    ```bash
    ./easyrsa revoke mykey # mykey代表自己的以.crt为后缀证书名字
    ```

## 配置openvpn

* 拷贝key文件到配置目录
    ```bash
    mkdir ~/openvpn-client      # 创建存放客户端秘钥的文件架

    # copy客户端key和配置到～/openvpn-client，配置文件在安装包里边的
    cp /opt/openvpn/easy-rsa/key/private/mysql.key ~/openvpn-client/
    cp /opt/openvpn/easy-rsa/key/issued/mysql.crt ~/openvpn-client/
    cp /opt/openvpn/easy-rsa/key/ca.crt ~/openvpn-client/
    cp /opt/openvpn/easy-rsa/key/ta.key ~/openvpn-client/
    cp /usr/local/src/openvpn-2.4.6/sample/sample-config-files/client.conf ~/openvpn-client/

    # copy服务端key到/etc/openvpn/server
    cp /opt/openvpn/easy-rsa/key/private/server.key /etc/openvpn/server/
    cp /opt/openvpn/easy-rsa/key/issued/server.crt /etc/openvpn/server/
    cp /opt/openvpn/easy-rsa/key/private/ca.key /etc/openvpn/server/
    cp /opt/openvpn/easy-rsa/key/ca.crt /etc/openvpn/server/
    cp /opt/openvpn/easy-rsa/key/ta.key /etc/openvpn/server/
    cp /opt/openvpn/easy-rsa/key/dh.pem /etc/openvpn/server/
    ```

### 服务端配置

* 打开服务端配置文件，并按照需要修改，把 “；”去掉即可启用
    ```sh
    vim /etc/openvpn/server/server.conf

    # 定义openvpn监听的IP地址，单网卡的可以不注明，但是多网卡的建议注明。
    ;local a.b.c.d

    # 定义openvpn监听的的端口，默认为1194端口。
    port 1194

    # 定义openvpn使用的协议，默认使用UDP。如果是生产环境的话，建议使用TCP协议。（二选一）
    ;proto tcp
    proto udp

    # 定义openvpn运行时使用哪一种模式，有两种运行模式一种是tap模式，一种是tun模式。（二选一）
    # tap模式也就是桥接模式，通过软件在系统中模拟出一个tap设备，该设备是一个二层设备，同时支持链路层协议
    # tun模式也就是路由模式，通过软件在系统中模拟出一个tun路由，tun是ip层的点对点协议。
    # 具体使用哪一种模式，需要根据自己的业务进行定义。
    ;dev tap
    dev tun

    # windwos系统如果有多个TAP-Win32 适配器，这里需要指定名称，也需要禁用TAP适配器的防火墙，其他系统不用操作
    ;dev-node MyTap

    # 定义openvpn使用的CA证书文件，该文件通过./easyrsa build-ca命令生成，CA证书主要用于验证客户证书的合法性。
    ca ca.crt

    # 定义openvpn服务器端使用的证书和key文件位置。
    cert server.crt
    key server.key  # 这个文件应该保密

    # 定义Diffie hellman文件位置
    dh dh.pem

    # 分配一个30位掩码的ip地址，除非需要支持很久的系统，不然不推荐开启
    ;topology subnet

    # 定义openvpn在使用tun路由模式时，分配给client端分配的IP地址段。
    server 192.168.250.0 255.255.255.0

    # 定义openvpn客户端和虚拟ip地址之间的关系。特别是在openvpn重启时,再次连接的客户端将依然被分配和断开之前的IP地址。
    ifconfig-pool-persist ipp.txt

    # 定义openvpn在使用tap桥接模式时，分配给客户端的IP地址段。
    ;server-bridge 192.168.250.0 255.255.255.0 10.8.0.50 192.168.250.1

    # 定义openvpn在使用tap桥接模式时，客户端从dhcp服务器获取ip地址，客户端配置，服务端无需配置  
    ;server-bridge

    # 向所有客户端推送的路由信息
    ;push "route 192.168.10.0 255.255.255.0"
    ;push "route 192.168.20.0 255.255.255.0"

    
    # 在/etc/openvpn/server/ccd目录下（如果没有手动创建）创建以客户端命名（证书的CN信息）的文件,来修改客户端的参数
    client-config-dir /etc/openvpn/server/ccd
    # 固定客户端的ip地址
    ifconfig-push 10.9.0.100
    # 如果客户端后边还有一个子网192.168.40.128,可以在文件中加入：
    iroute 192.168.40.128 255.255.255.248   # 客户端后边的子网网段
    # 给此客户客户端推送路由
    push "192.168.253.0 255.255.255.0"
    # 以上三条命令是在ccd目录下当前客户端名称的文件配置的

    # 服务端添加去客户端后边子网的路由
    route 192.168.40.128 255.255.255.248

    # 假设您希望为不同的客户端组启用不同的防火墙访问策略。有两种方法：
    # (1)运行多个OpenVPN进程，每个组一个，并为每个进程添加一个防火墙TUN/TAP接口。
    # (2)（推荐）创建一个脚本来动态修改防火墙以响应来自不同客户端的访问，脚本不会
    ;learn-address ./script

    # 这条命令可以重定向客户端的网关，在进行FQ时会使用到。也就是修改默认网关到vpn
    ;push "redirect-gateway def1 bypass-dhcp"

    # 向客户端推送的DNS信息。
    ;push "dhcp-option DNS 208.67.222.222"
    ;push "dhcp-option DNS 208.67.220.220"

    # 这条命令可以使客户端之间能相互访问，默认设置下客户端间是不能相互访问的。
    client-to-client

    # 定义openvpn一个证书在同一时刻是否允许多个客户端接入，默认没有启用。
    ;duplicate-cn

    # 定义活动连接保时期限，以下配置的意思为：每隔10秒接收一次类似ping的消息，超过120秒没接收到，即判定对端下线
    keepalive 10 120

    # 为了加强SSL/TLS提供的额外安全性，创建一个“HMAC防火墙”来帮助阻止DoS攻击和UDP端口泛滥。
    # 通过 openvpn --genkey --secret ta.key 命令生成ta.key
    # 服务器和每个客户端必须有此密钥。“0”代表服务器，“1”代表客户端。
    tls-auth ta.key 0 # 这个文件应该保密

    # 保证客户端和服务端使用一样，用于加密密码
    cipher AES-256-CBC

    # 启用lz4-v2压缩，并将选项推送到客户端，
    ;compress lz4-v2
    ;push "compress lz4-v2"

    # 与老客户端兼容的压缩，客户端配置文件也需要有这项。与上边的二选一
    ;comp-lzo

    # 定义最大客户端并发连接数量
    ;max-clients 100

    # 定义openvpn运行时使用的用户及用户组。
    user nobody
    group nobody

    # 通过keepalive检测超时后，重新启动VPN，不重新读取keys，保留第一次使用的keys。
    persist-key

    # 通过keepalive检测超时后，重新启动VPN，一直保持tun或者tap设备是linkup的。否则网络连接，会先linkdown然后再linkup。
    persist-tun

    # 把openvpn的一些状态信息写到文件中，比如客户端获得的IP地址。
    status /var/log/openvpn/server/openvpn-status.log

    # 记录日志，每次重新启动openvpn后删除原有的log信息。也可以自定义log的位置。默认是在/etc/openvpn/目录下。
    log         /var/log/openvpn/server/openvpn.log

    # 记录日志，每次重新启动openvpn后追加原有的log信息。
    ;log-append  /var/log/openvpn/server/openvpn.log

    # 设置日志记录冗长级别。
    verb 3

    # 重复日志记录限额
    ;mute 20

    # 通知客户端，当服务器重新启动时，它可以自动重新连接。使用tcp协议的时候这里需要注释掉
    explicit-exit-notify 1

    #########################################################################################
    #                                                                                       #
    #                        下面的配置为自定义配置，标准配置文档里边没有                           #
    #                                                                                       #
    #########################################################################################

    ```
* 启用服务
    ```sh
    systemctl start openvpn@server  # 启动服务
    systemctl enable openvpn@server # 开机启动
    firewall-cmd --add-service=openvpn --permanent --zone=public # 开启ip伪装
    firewall-cmd --permanent --direct --passthrough ipv4 -t nat -I POSTROUTING -o eth0 -j MASQUERADE -s 192.168.250.0/24 # 设置ip伪装
    firewall-cmd --reload # 设置生效
    echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf # 打开IP转发
    systemctl restart network
    ```

### 客户端配置

* 打开客户端配置文件，并按照需要修改，把 “；”去掉即可启用
    ```sh
    vim ~/openvpn/openvpn-client/client.conf

    # 定义这是一个client，配置从server端pull拉取过来，如IP地址，路由信息之类，Server使用push指令推送过来。
    client

    # 定义openvpn运行的模式，这个地方需要严格和Server端保持一致。
    dev tun
    ;dev tap

    # windwos系统如果有多个TAP-Win32适配器，这里需要指定名称，也需要禁用TAP适配器的防火墙，其他系统不用操作
    ;dev-node MyTap

    # 定义openvpn使用的协议，这个地方需要严格和Server端保持一致。
    proto udp
    ;proto tcp

    # 设置Server的IP地址和端口，这个地方需要严格和Server端保持一致。
    # 如果有多台机器做负载均衡，可以多次出现remote关键字。
    remote www.net-cc.com 1194
    ;remote my-server-2 1194

    # 随机选择一个Server连接，否则按照顺序从上到下依次连接。该选项默认不启用。
    ;remote-random

    # 始终重新解析Server的IP地址（如果remote后面跟的是域名），保证Server IP地址是动态的使用DDNS动态更新DNS后，
    # Client在自动重新连接时重新解析Server的IP地址。这样无需人为重新启动，即可重新接入VPN。
    resolv-retry infinite

    # 定义在本机不邦定任何端口监听incoming数据。
    nobind

    # 定义openvpn运行时使用的用户及用户组。
    ;user nobody
    ;group nobody

    # 尝试在重新启动时保留一些状态。
    persist-key
    persist-tun

    # http代理配置，如果需要代理认证，请参照手册更改，具体没研究
    ;http-proxy-retry # 关于连接失败的重试
    ;http-proxy [proxy server] [proxy port #]

    # 无线网络常常会产生大量的重复数据包，开启可关闭此告警
    ;mute-replay-warnings

    # 定义CA证书的文件名，用于验证Server CA证书合法性，该文件一定要与服务器端ca.crt是同一个文件。
    ca ca.crt

    # 定义客户端的证书文件。
    cert client.crt

    # 定义客户端的密钥文件。
    key client.key

    # Server使用build-key-server脚本生成的，在x509 v3扩展中加入了ns-cert-type选项。防止client使用他们的keys ＋ DNS hack欺骗vpn client连接他们假冒的VPN Server，因为他们的CA里没有这个扩展。
    remote-cert-tls server

    # 为了加强SSL/TLS提供的额外安全性，创建一个“HMAC防火墙”来帮助阻止DoS攻击和UDP端口泛滥。
    # 通过 openvpn --genkey --secret ta.key 命令生成ta.key
    # 服务器和每个客户端必须有此密钥。“0”代表服务器，“1”代表客户端。
    tls-auth ta.key 1

    # 保证客户端和服务端使用一样，用于加密密码
    cipher AES-256-CBC

    # 如果服务器启用了comp-lzo压缩，此处需要开启
    ;comp-lzo

    # 设置日志记录冗长级别。
    verb 3

    # 重复日志记录限额
    ;mute 20

    #########################################################################################
    #                                                                                       #
    #                        下面的配置为自定义配置，标准配置文档里边没有                          #
    #                                                                                       #
    #########################################################################################

    # 不在内存里边保存账号秘钥和秘钥
    auth-nocache

    ```

## 开启简单密码认证

> 首先按照前边的已经配置好了，用证书验证测试也可以通过了，可以更换成密码认证或者密码证书双认证

### 创建认证脚本

* 创建文件,并修改权限，psw-file的权限修改最小
    ```sh
    mkdir -p /etc/openvpn/openvpn-passwd
    cd /etc/openvpn/openvpn-passwd
    touch checkpsw.sh openvpn-password.log psw-file
    touch /var/log/openvpn/server/openvpn-password.log
    chmod 400 psw-file | chown nobody:nobody psw-file
    chmod u+x checkpsw.sh
    ```
* 编辑checkpsw.sh 文件，添加一下内容
    ```sh
    #!/bin/sh
    ###########################################################
    # checkpsw.sh (C) 2004 Mathias Sundman <mathias@openvpn.se>
    #
    # This script will authenticate OpenVPN users against
    # a plain text file. The passfile should simply contain
    # one row per user with the username first followed by
    # one or more space(s) or tab(s) and then the password.

    PASSFILE="/etc/openvpn/openvpn-passwd/psw-file"
    LOG_FILE="/var/log/openvpn/server/openvpn-password.log"
    TIME_STAMP=`date "+%Y-%m-%d %T"`

    ###########################################################

    if [ ! -r "${PASSFILE}" ]; then
    echo "${TIME_STAMP}: Could not open password file \"${PASSFILE}\" for reading." >> ${LOG_FILE}
    exit 1
    fi

    CORRECT_PASSWORD=`awk '!/^;/&&!/^#/&&$1=="'${username}'"{print $2;exit}' ${PASSFILE}`

    if [ "${CORRECT_PASSWORD}" = "" ]; then 
    echo "${TIME_STAMP}: User does not exist: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
    exit 1
    fi

    if [ "${password}" = "${CORRECT_PASSWORD}" ]; then 
    echo "${TIME_STAMP}: Successful authentication: username=\"${username}\"." >> ${LOG_FILE}
    exit 0
    fi

    echo "${TIME_STAMP}: Incorrect password: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
    exit 1
    ```

### 修改配置文件

* 服务端
    ```sh
    #############################  密码认证部分（不需要密码的可以不填）  ##########################

    # 加入脚本处理，如密码认证
    script-security 3

    # 开启密码认证，指定使用的认证脚本
    auth-user-pass-verify /etc/openvpn/openvpn-passwd/checkpsw.sh via-env

    # 则代表只使用用户名密码方式验证登录，可以吧前边的crt和key注释掉
    client-cert-not-required

    # 使用客户提供的UserName作为Common Name
    username-as-common-name

    #########################################################################################

    ```
* 客户端
    ```sh

    #############################  密码认证部分（不需要密码的可以不填）  ##########################

    # 开启密码认证，如果注释掉上边的key和cert文件的话就只有密码认证啦，并没有双重认证
    auth-user-pass

    #########################################################################################

    ```
* 创建用户账号
    修改/etc/openvpn/openvpn-passwd/psw-file文件，格式如下：
    ```sh
    zhangcc 123456
    mysql 123456
    ```
* 重启服务
    ```sh
    systemctl restart openvpn@server
    ```
* 配置客户端自动输入密码登录
    在配置文件目录里新建passwd.txt文件，编辑这个文件，并在第一行输入账号，第二行输入密码，如下：
    ```sh
    username
    password
    ```
    编辑.ovpn或者.conf配置文件，修改 auth-user-pass
    ```sh
    auth-user-pass passwd.txt
    ```
    然后再次连接的时候就不需要账号密码了