---
title: "openssl证书管理"
date: 2020-09-06T18:37:29+08:00
lastmod: 2020-09-24T13:44:07+08:00
draft: false
keywords: ['SSL证书','https证书']
description: "使用openssl生成支持SAN扩展的证书笔记"
tags: ['PKI']
categories: ['linux']
---

OpenSSL 是一个免费开源的库，它提供了构建数字证书的命令行工具，其中一些可以用来自建 Root CA。
<!--more-->



# 安装openssl

```bash
sudo yum install openssl openssl-libs
```

# 自建Root CA

首先找到一个放置证书的文件夹，比如 ~/ca 下，下方的测试也在改目录下，如果你要更换其他目录，记得替换下文中的目录地址

## 设定文件架构，并配置openssl设置

1. 创建目录

    ```bash
    mkdir -p ~/ca/{certs,crl,newcerts,private}
    cd ~/ca
    chmod 700 private
    touch index.txt
    echo 1000 > serial
    cp /etc/pki/tls/openssl.cnf ./
    ```

2. 修改openssl.cnf配置

    ```ini
    # 更改openssl.cnf的以下参数
    [ CA_default ]
    dir                   = /home/cc/ca
    certs                 = $dir/certs
    certificate           = $dir/certs/cacert.pem
    crl                   = $dir/crl/cacrl.pem
    private_key           = $dir/private/cakey.pem
    crl_dir               = $dir/crl
    database              = $dir/index.txt
    new_certs_dir         = $dir/newcerts
    serial                = $dir/serial
    crlnumber             = $dir/crlnumber

    # 在openssl.cnf的末尾添加以下配置    
    [ v3_intermediate_ca ]
    # Extensions for a typical intermediate CA (`man x509v3_config`).
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid:always,issuer
    basicConstraints = critical, CA:true, pathlen:0
    keyUsage = critical, digitalSignature, cRLSign, keyCertSign
    ```

## 创建Root CA Key

```bash
# 注意，此处提供了-aes256参数用来加密密钥，所以必须输入密码
cd ~/ca
openssl genrsa -aes256 -out private/cakey.pem 4096
chmod 400 private/cakey.pem
```

## 创建Root CA cert

```bash
cd ~/ca
# 需要输入root CA key的密码
openssl req -config openssl.cnf  -key private/cakey.pem  -new -x509 -days 14500 -sha256 -extensions v3_ca   -out certs/cacert.pem
chmod 444 certs/cacert.pem
```

**以上命令需要提供CA Key的密码，就是刚刚创建CA key的密码，还需要提供Country Name(C)、State(ST)、Locality(L)、Organization(O)、Organizational Unit(OU)、Common Name(CN)、电子邮件地址(emailAddress )，注意：Common Name不可以重复，例：**

```text
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:ShangHai
Locality Name (eg, city) [Default City]:ShangHai
Organization Name (eg, company) [Default Company Ltd]:Net Cc
Organizational Unit Name (eg, section) []:Net Cc Certificate Authority
Common Name (eg, your name or your server's hostname) []:Net Cc Root CA
Email Address []:
```

## 验证证书

* 验证命令

    ```bash
    $ openssl x509 -noout -text -in certs/ca.cert
    ```

    主要看以下参数是否正确：`数字签名（Signature Algorithm）`、`有效时间（Validity）`、`主体（Issuer）`、`公钥（Public Key）`、`X509v3 扩展，openssl config 中配置了 v3_ca，所以会生成此项`

# 自建Intermediate CA

## 设定文件架构，并配置openssl设置

1. 创建目录
   
    ```bash
    mkdir -p intermediate/{certs,crl,newcerts,private,csr}
    cd ~/ca/intermediate
    chmod 700 private
    touch index.txt
    echo 1000 > serial
    echo 1000 > crlnumber
    cp ../openssl.cnf ./
    ```

2. 修改openssl.cnf配置

    ```ini
    # 更改openssl.cnf的以下配置
    [ CA_default ]
    dir              = /home/cc/ca/intermediate
    [ policy_match ]
    organizationName        = supplied
    ```

## 创建Intermediate CA Key

```bash
cd ~/ca/intermediate
# 注意，此处提供了-aes256参数用来加密密钥，所以必须输入密码，等会生成CA证书和签名的时候需要用到这个密码
openssl genrsa -aes256 -out private/cakey.pem 4096
chmod 400 private/cakey.pem
```

## 创建Intermediate CA CSR，注意 Common Name 不要与 Root CA 的一样

```bash
cd ~/ca/intermediate
# 需要输入intermediate CA key的密码
openssl req -config openssl.cnf -new -sha256 -key private/cakey.pem -out csr/ca.csr
```

**以上命令需要提供Intermediate CA Key的密码和一些名字，注意：Common Name不可以重复，例：**

```text
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:ShangHai
Locality Name (eg, city) [Default City]:ShangHai
Organization Name (eg, company) [Default Company Ltd]:Net Cc
Organizational Unit Name (eg, section) []:Net Cc Certificate Authority
Common Name (eg, your name or your server's hostname) []:Net Cc Intermediate CA
Email Address []:
```

## 使用root ca进行签名

```bash
cd ~/ca
# 使用root CA的openssl.cnf文件，需要输入root CA 密钥的密码
openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 7300 -notext -md sha256 -in intermediate/csr/ca.csr -out intermediate/certs/cacert.pem
chmod 444 intermediate/certs/cacert.pem
```

## 验证

```bash
cd ~/ca/intermediate
openssl x509 -noout -text -in certs/cacert.pem
openssl verify -CAfile ../certs/cacert.pem  certs/cacert.pem
```

## 创建证书链，将 root cert 和 intermediate cert 合并到一起

```bash
cd ~/ca/intermediate
cat ../certs/cacert.pem certs/cacert.pem > certs/ca-chaincert.pem
# 验证证书链
openssl verify -CAfile certs/ca-chaincert.pem  certs/cacert.pem
```

# 创建带有SAN扩展的证书


## 创建san.cnf配置文件

```ini
[ req_san ]
subjectKeyIdentifier = hash
basicConstraints = CA:FALSE
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth,serverAuth
subjectAltName = @alt_names # SAN调用了alt_names所以我们下边要添加一个alt_names
# SAN地址设置，DNS是域名，ip是IP地址
[ alt_names ]
DNS.1 = net-cc.com
DNS.2 = *.net-cc.com
DNS.3 = localhost
IP.1 = 127.0.0.1
```

## 创建server/client Key

```bash
cd ~/ca/intermediate/
# 此处我们没有添加加密密码，如果有加密的话，服务器可能无法解密使用key验证。。
openssl genrsa -out private/net-cc.com.key.pem 2048
chmod 400 private/net-cc.com.key.pem
```
		
## 创建证书请求文件CSR

```bash
cd ~/ca/intermediate/
openssl req -config openssl.cnf -key private/net-cc.com.key.pem  -new -sha256 -out csr/net-cc.com.csr
```

**需要提供Country Name(C)、State(ST)、Locality(L)、Organization(O)、Organizational Unit(OU)、Common Name(CN)、电子邮件地址(emailAddress )，注意：Common Name不可以重复，例：**

```text
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:ShangHai
Locality Name (eg, city) [Default City]:ShangHai
Organization Name (eg, company) [Default Company Ltd]:net-cc.com
Organizational Unit Name (eg, section) []:www.net-cc.com
Common Name (eg, your name or your server's hostname) []:net-cc.com
Email Address []:
```

## 使用Intermediate CA进行签名

```bash
cd ~/ca/intermediate/
openssl ca -config openssl.cnf -extfile san.cnf -extensions req_san -days 3650 -notext -md sha256 -in csr/net-cc.com.csr -out certs/net-cc.com.cert.pem
chmod 444 certs/net-cc.com.cert.pem
```

## 验证

```bash
openssl verify -CAfile certs/ca-chaincert.pem certs/net-cc.com.cert.pem
openssl x509 -noout -text -in certs/net-cc.com.cert.pem
```
	
## Server/Client的证书密钥文件位置

* key:  `~/ca/intermediate/private/net-cc.com.key.pem`
* cert: `~/ca/intermediate/certs/net-cc.com.cert.pem`
* CA Cert: `~/ca/intermediate/certs/ca-chaincert.pem`
