---
title: "二进制方式安装K8S"
date: 2020-09-20T01:37:56+08:00
lastmod: 2020-12-05T22:04:04+08:00
draft: false
keywords: ['k8s', '安装k8s','k8s集群','容器']
description: "通过二进制方式安装k8s高可用集群"
tags: ['k8s']
categories: ['k8s']
autoCollapseToc: true
---

<!--more-->

# 节点信息

| IP地址         | 主机名称 | DNS记录 | 类型                   | 系统    |
| -------------- | -------- | ------- | ---------------------- | ------- |
| 192.168.122.10 | node1    | node1   | Worker Node/etcd       | centos7 |
| 192.168.122.11 | node2    | node2   | Worker Node            | centos7 |
| 192.168.122.12 | node3    | node3   | Master Node/etcd/nginx | centos8 |
| 192.168.122.13 | node4    | node4   | Master Node/etcd/nginx | centos8 |
| 192.168.122.2  |          | k8s.api | vrrp                   |         |
| 10.110.0.0/16  |          |         | k8s service            |         |
| 10.120.0.0/16  |          |         | k8s pod                |         |

电脑配置较低，这么多虚拟机跑不动，所以这一套高可用集群分两部分实施，先部署一套单Master架构（192.168.122.12），再扩容为多Master架构（上述规划），顺便熟悉下Master扩容流程

## 初始化节点

尽管主节点中包含etcd端口，但您也可以在外部或自定义端口上托管您自己的etcd集群

```sh
# 节点之间免密登录
ssh-keygen -b 2048 -t rsa
ssh-copy-id root@node1
ssh-copy-id root@node2
ssh-copy-id root@node3
ssh-copy-id root@node4
scp ~/.ssh/id_rsa node1:~/.ssh/
scp ~/.ssh/id_rsa node2:~/.ssh/
scp ~/.ssh/id_rsa node3:~/.ssh/
scp ~/.ssh/id_rsa node4:~/.ssh/

# --------------以下所有节点都要设置-------------------
# 关闭swap, 我没关，苦逼，内存不够
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久
# 根据规划设置主机名
hostnamectl set-hostname <hostname>
# 添加dns解析记录
cat >> /etc/hosts << EOF
192.168.122.2  k8s.api
192.168.122.10 node1
192.168.122.11 node2
192.168.122.12 node3
192.168.122.13 node4
EOF
# 时间同步
sed -i '3i server ntp.aliyun.com iburst' /etc/chrony.conf
systemctl restart chronyd.service
# 创建程序目录（所有节点）
mkdir -p /opt/kubernetes/{bin,etc,ssl,logs,data}
mkdir -p /opt/kubernetes/etc/conf.d

# --------------以下只设置Master Node-------------------
# 创建用户
sudo groupadd -r kube
sudo useradd  -s /sbin/nologin -M -d / -g kube -c "Kubernetes user" -r -K MAIL_DIR=/dev/null kube
# 设置防火墙
firewall-cmd --permanent --add-port=6443/tcp # kube-api 安全端口
firewall-cmd --permanent --add-port=10250-10252/tcp
firewall-cmd --reload

# --------------以下只设置Wroker Node-------------------
# 设置防火墙
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --permanent --add-port=30000-32767/udp
firewall-cmd --permanent --service=http
firewall-cmd --permanent --service=https
firewall-cmd --reload
## briger流量转到iptables模块
echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf
## 生效
systemctl restart systemd-modules-load.service

# 开启ipvs
## 安装ipset,一般默认都有
yum install ipset -y
## 为了方便查看ipvs规则我们要安装ipvsadm(可选)
yum install ipvsadm -y

# 将桥接的IPv4流量传递到iptables的链（所有Worker Node）
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system  # 生效
```
防火墙放行的端口介绍：

| 协议 | 方向 | 港口范围     | 目的                    | 使用               |
| ---- | ---- | ------------ | ----------------------- | ------------------ |
| TCP  | 入站 | 6443*，8080* | Kubernetes API server   | 所有               |
| TCP  | 入站 | 2379-2380    | etcd server client API  | etcd               |
| TCP  | 入站 | 10250        | Kubelet API             | 本机，控制平面组件 |
| TCP  | 入站 | 10251        | kube-scheduler          | 本机               |
| TCP  | 入站 | 10252        | kube-controller-manager | 本机               |
| TCP  | 入站 | 30000-32767  | NodePort Services**     | 所有node节点       |

**NodePort Services的默认端口范围。**

**使用 * 标记的端口号都可以自定义的，所以您需要保证所自定义的端口是开放的。**


# 安装ETCD集群

参考 [etcd安装配置]({{< relref "etcd install.md" >}}) , 拷贝客户端证书到kubernetes server节点的`/opt/kubernetes/ssl/`目录下；

我的客户端证书：`etcd-client.crt` `etcd-client.key`  `etcd-ca.crt`

# PKI证书管理

可以参考官方文档：<https://kubernetes.io/zh/docs/setup/best-practices/certificates/>来生成证书

**注意： 证书我是在server1（192.168.122.12）上进行生成管理的**

**证书的O和CN字段在RBAC授权的时候被认为是`组`和`用户`,所以请不要随意更改，不然RBAC预定义角色无法匹配，导致授权失败**

## 创建CA

另外还需要获取用于服务账户管理的密钥对，也就是sa.key和sa.pub。

| 路径                   | 默认 CN                   | 描述                   |
| ---------------------- | ------------------------- | ---------------------- |
| kube-ca.crt,key        | kubernetes-ca             | Kubernetes 通用 CA     |
| front-proxy-ca.crt,key | kubernetes-front-proxy-ca | 用于 [前端代理][proxy] |

* kubernetes-ca

  ```sh
  mkdir -p ~/ca/{kube-ca,kube-proxy-ca}
  cd ~/ca/kube-ca
  openssl genrsa -out kube-ca.key  2048

  openssl req -extensions v3_ca \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=kubernetes-ca/emailAddress=admin@admin.com" \
  -new -x509 -days 14500 -sha256 \
  -key=kube-ca.key -out kube-ca.crt

  # 创建证书签名用的配置文件
  cat > kubernetes-ca.conf<<EOF
  # 用来对服务端证书签名
  [ req_server ]
  subjectKeyIdentifier = hash
  basicConstraints = CA:FALSE
  authorityKeyIdentifier = keyid,issuer:always
  keyUsage = critical, digitalSignature, keyEncipherment
  extendedKeyUsage = clientAuth,serverAuth
  subjectAltName = @alt_names

  # dns中kubernetes开头的是APIserver需要用到的记录
  # 要包含所有master节点和负载均衡的IP地址、DNS记录
  [ alt_names ]
  DNS.1 = kubernetes
  DNS.2 = kubernetes.default
  DNS.3 = kubernetes.default.svc
  DNS.4 = kubernetes.default.svc.cluster
  DNS.5 = kubernetes.default.svc.cluster.local
  DNS.6 = localhost
  DNS.7 = node3
  DNS.8 = node4
  DNS.9 = k8s.api
  IP.1 = 192.168.122.12
  IP.2 = 192.168.122.13
  IP.3 = 192.168.122.2
  IP.4 = 127.0.0.1
  IP.5 = 10.110.0.1

  # 用来对客户端证书签名
  [ req_client ]
  subjectKeyIdentifier = hash
  basicConstraints = CA:FALSE
  authorityKeyIdentifier = keyid,issuer:always
  keyUsage = critical, digitalSignature, keyEncipherment
  extendedKeyUsage = clientAuth
  EOF

  # 更改权限
  chmod 444 kube-ca.crt
  chmod 400 kube-ca.key
  ```

* kubernetes-front-proxy-ca

  ```sh
  cd ~/ca/kube-proxy-ca
  openssl genrsa -out front-proxy-ca.key 2048

  openssl req -extensions v3_ca \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=kubernetes-front-proxy-ca/emailAddress=admin@admin.com" \
  -new -x509 -days 14500 -sha256 \
  -key=front-proxy-ca.key -out front-proxy-ca.crt

  # 创建证书签名用的配置文件
  cat > front-proxy-ca.conf<<EOF
  # 用来对客户端证书签名
  [ req_client ]
  subjectKeyIdentifier = hash
  basicConstraints = CA:FALSE
  authorityKeyIdentifier = keyid,issuer:always
  keyUsage = critical, digitalSignature, keyEncipherment
  extendedKeyUsage = clientAuth
  EOF

  # 更改证书/密钥权限为只读
  chmod 444 front-proxy-ca.crt
  chmod 400 front-proxy-ca.key
  ```

## Server证书

两台api server我共用了证书，所以SAN的`hostname`,`host_ip`,`advertise_ip`我写了所有主机的名字;

| 默认 CN                       | 父级 CA                   | O (位于 Subject 中) | 类型   | 主机 (SAN)                                  |
| ----------------------------- | ------------------------- | ------------------- | ------ | ------------------------------------------- |
| kube-apiserver                | kubernetes-ca             |                     | server | [hostname], [Host_IP], [advertise_IP], 注:1 |
| kube-apiserver-kubelet-client | kubernetes-ca             | system:masters      | client |                                             |
| front-proxy-client            | kubernetes-front-proxy-ca |                     | client |                                             |

注1: kubeadm为负载均衡所使用的固定IP或 NS名: kubernetes、kubernetes.default、kubernetes.default.svc、 kubernetes.default.svc.cluster、kubernetes.default.svc.cluster.local

* apiserver证书

  ```sh
  mkdir -p ~/ca/kube-ca/{apiserver,kubeconfig,kubelet}
  cd ~/ca/kube-ca/apiserver

  # 创建密钥
  openssl genrsa -out apiserver.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=kube-apiserver/emailAddress=admin@admin.com" \
  -key apiserver.key -out apiserver.csr

  # 使用kubernetes-ca进行签名获取带有SAN扩展的证书
  openssl x509 -in apiserver.csr -out apiserver.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile ../kubernetes-ca.conf -extensions req_server

  # 权限
  rm -f apiserver.csr
  chmod 400 apiserver.key
  chmod 444 apiserver.crt
  ```

* apiserver-kubelet-client证书

  apiserver访问kubelet的证书密钥对（RBAC会针对`system:masters`用户进行授权）

    ```sh
    cd ~/ca/kube-ca/apiserver
    # 创建密钥
    openssl genrsa -out apiserver-kubelet-client.key 2048

    # 创建证书请求文件
    openssl req -new -sha256 \
    -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=system:masters/emailAddress=admin@admin.com" \
    -key apiserver-kubelet-client.key -out apiserver-kubelet-client.csr

    # 使用kubernetes-ca进行签名
    openssl x509 -in apiserver-kubelet-client.csr -out apiserver-kubelet-client.crt \
    -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
    -req -sha256 -days 7300 -extfile ../kubernetes-ca.conf -extensions req_client

    rm -f apiserver-kubelet-client.csr
    chmod 400 apiserver-kubelet-client.key
    chmod 444 apiserver-kubelet-client.crt
    ```

* kubelet服务证书

  kubelet也启用https了，所以要给他生成一个证书，外部访问kubelet的的程序可以根据kube-ca进行验证kubelet的服务证书，例如安装metrics-server性能指标监控程序的时候，他需要验证kubelet证书

  注意：DNS和IP要写所有的安装kubelet节点的ip和主机名称

  ```sh
  cd ~/ca/kube-ca/kubelet
  # 创建密钥
  openssl genrsa -out kubelet.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=kube-kubelet/emailAddress=admin@admin.com" \
  -key kubelet.key -out kubelet.csr

  # 创建证书签名用的配置文件
  cat > kubelet-cert.conf<<EOF
  # 用来对服务端证书签名
  [ req_server ]
  subjectKeyIdentifier = hash
  basicConstraints = CA:FALSE
  authorityKeyIdentifier = keyid,issuer:always
  keyUsage = critical, digitalSignature, keyEncipherment
  extendedKeyUsage = serverAuth
  subjectAltName = @alt_names

  # 要包含所有worker节点的IP地址、DNS记录
  [ alt_names ]
  DNS.1 = localhost
  DNS.2 = node1
  DNS.3 = node2
  IP.1 = 192.168.122.10
  IP.2 = 192.168.122.11
  IP.3 = 127.0.0.1
  EOF

  # 使用kubernetes-ca进行签名获取带有SAN扩展的证书
  openssl x509 -in kubelet.csr -out kubelet.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile kubelet-cert.conf -extensions req_server

  # 权限
  rm -f kubelet.csr
  chmod 400 kubelet.key
  chmod 444 kubelet.crt
  ```


* front-proxy-client证书

  只有当你运行kube-proxy并要支持[扩展API服务器](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/setup-extension-api-server/)时，才需要front-proxy证书

  ```sh
  mkdir ~/ca/kube-proxy-ca/front-proxy-client
  cd ~/ca/kube-proxy-ca/front-proxy-client
  # 创建密钥
  openssl genrsa -out front-proxy-client.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=front-proxy-client/emailAddress=admin@admin.com" \
  -key front-proxy-client.key -out front-proxy-client.csr 

  # 使用中间CA进行签名
  openssl x509 -in front-proxy-client.csr -out front-proxy-client.crt \
  -CA ../front-proxy-ca.crt -CAkey ../front-proxy-ca.key -CAcreateserial \
  -req -sha256 -days 7300 -extfile ../front-proxy-ca.conf -extensions req_client

  rm -f front-proxy-client.csr
  chmod 400 front-proxy-client.key
  chmod 444 front-proxy-client.crt
  ```

* 服务账户管理的密钥对

  ```sh
  cd ~/ca/kube-ca/apiserver
  openssl genrsa -out sa.key 2048
  openssl rsa -in sa.key -pubout -out sa.pub
  chmod 400 sa.key
  chmod 444 sa.pub
  ```

## 用户证书

用户证书是kubectl、kubelet、kube-controller-manager、kube-scheduler等程序连接API server时候用到的客户端证书，可以直接写入配置文件，不需要拷贝证书/密钥到节点上，

使用kubectl创建连接apiserver的配置文件，等配置生成完后复制`admin.conf`到`~/.kube/config`, 然后删除`KUBECONFIG`的环境变量即可恢复kubectl对apiserver的访问


* 配置文件信息

  | 文件名                  | 命令                    | 凭据名称                   | 默认 CN                             | O (位于 Subject 中) | 说明                                                         |
  | ----------------------- | ----------------------- | -------------------------- | ----------------------------------- | ------------------- | ------------------------------------------------------------ |
  | admin.conf              | kubectl                 | default-admin              | kubernetes-admin                    | system:masters      | 配置集群的管理员                                             |
  | kubelet.conf            | kubelet                 | default-auth               | system:node:[nodeName] （参阅注释） | system:nodes        | 集群中的每个节点都需要一份,配置了连接API Server等的参数      |
  | controller-manager.conf | kube-controller-manager | default-controller-manager | system:kube-controller-manager      |                     | 必需添加到 `manifests/kube-controller-manager.yaml` 清单中   |
  | scheduler.conf          | kube-scheduler          | default-scheduler          | system:kube-scheduler               |                     | 必需添加到 `manifests/kube-scheduler.yaml` 清单中            |
  | proxy.conf              | kube-proxy              | default-proxy              | system:kube-proxy                   |                     | 集群中的每个节点都需要一份,配置了连接API Server等的参数proxy |


  注释：kubelet.conf中[nodeName]的值必须与kubelet向apiserver注册时提供的节点名称的值完全匹配。 有关更多详细信息，请阅读[节点授权](https://kubernetes.io/zh/docs/reference/access-authn-authz/node/)。

* 安装kubectl

  k8s集群管理工具

  ```sh
  cd /tmp
  wget https://dl.k8s.io/v1.19.1/kubernetes-client-linux-amd64.tar.gz
  tar -zxf kubernetes-client-linux-amd64.tar.gz
  cp kubernetes/client/bin/kubectl /usr/bin/
  rm -rf kubernetes*
  # kubectl命令补全
  echo 'eval "$(kubectl completion bash)"' >> ~/.bashrc
  source ~/.bashrc
  ```

* admin.conf

  kubectl客户端连接apiserver时候使用的配置文件

  kubectl客户端默认调用`~/.kube/config`的配置文件
  
  可以通过`KUBECONFIG`变量来控制

  ```sh
  mkdir ~/ca/kube-ca/kubeconfig
  cd ~/ca/kube-ca/kubeconfig
  # 创建密钥
  openssl genrsa -out kube-admin.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=system:masters/OU=net-cc.com/CN=kubernetes-admin/emailAddress=admin@admin.com" \
  -key kube-admin.key -out kube-admin.csr

  # 使用中间CA进行签名
  openssl x509 -in kube-admin.csr -out kube-admin.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile ../kubernetes-ca.conf -extensions req_client

  rm -f kube-admin.csr
  chmod 400 kube-admin.key
  chmod 444 kube-admin.crt

  # 把证书直接写道配置文件中，客户端不用保留证书了，(`--embed-certs`控制)
  export KUBECONFIG=admin.conf
  kubectl config set-cluster default-cluster --server=https://node3:6443 --certificate-authority ../kube-ca.crt --embed-certs
  kubectl config set-credentials default-admin --client-key kube-admin.key --client-certificate kube-admin.crt --embed-certs
  kubectl config set-context default-system --cluster default-cluster --user default-admin
  kubectl config use-context default-system
  ```

* kubelet.conf

  两个Worker节点，node1我使用手动生成证书的方式

  node2我使用自动签发证书的方式，所以这里未生成证书

  ```sh
  cd ~/ca/kube-ca/kubeconfig
  export NODENAME=node1
  # 创建密钥
  openssl genrsa -out kube-kubelet-$NODENAME.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=system:nodes/OU=net-cc.com/CN=system:node:$NODENAME/emailAddress=admin@admin.com" \
  -key kube-kubelet-$NODENAME.key -out kube-kubelet-$NODENAME.csr

  # 使用kubernetes-ca进行签名
  openssl x509 -in kube-kubelet-$NODENAME.csr -out kube-kubelet-$NODENAME.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile ../kubernetes-ca.conf -extensions req_client

  rm -f kube-kubelet-$NODENAME.csr
  chmod 400 kube-kubelet-$NODENAME.key
  chmod 444 kube-kubelet-$NODENAME.crt

  # 把证书直接写道配置文件中，客户端不用保留证书了，(`--embed-certs`控制)
  export KUBECONFIG=kubelet-$NODENAME.conf
  kubectl config set-cluster default-cluster --server=https://node3:6443 --certificate-authority ../kube-ca.crt --embed-certs
  kubectl config set-credentials default-auth --client-key kube-kubelet-$NODENAME.key --client-certificate kube-kubelet-$NODENAME.crt --embed-certs
  kubectl config set-context default-system --cluster default-cluster --user default-auth	
  kubectl config use-context default-system
  ```

* proxy.conf

  所有Worker节点的kube-proxy程序可以共用这一个配置文件，所以生成一次即可

  注意：证书的CN不要变更

  ```sh
  cd ~/ca/kube-ca/kubeconfig
  # 创建密钥
  openssl genrsa -out kube-proxy.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=system:kube-proxy/emailAddress=admin@admin.com" \
  -key kube-proxy.key -out kube-proxy.csr

  # 使用kubernetes-ca进行签名
  openssl x509 -in kube-proxy.csr -out kube-proxy.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile ../kubernetes-ca.conf -extensions req_client

  rm -f kube-proxy.csr
  chmod 400 kube-proxy.key
  chmod 444 kube-proxy.crt

  # 把证书直接写道配置文件中，客户端不用保留证书了，(`--embed-certs`控制)
  cd ~/ca/kube-ca/kubeconfig
  export KUBECONFIG=proxy.conf
  kubectl config set-cluster default-cluster --server=https://node3:6443 --certificate-authority ../kube-ca.crt --embed-certs
  kubectl config set-credentials default-proxy --client-key kube-proxy.key --client-certificate kube-proxy.crt --embed-certs
  kubectl config set-context default-system --cluster default-cluster --user default-proxy
  kubectl config use-context default-system
  ```

* controller-manager.conf

  注意：证书的CN不要变更

  ```sh
  cd ~/ca/kube-ca/kubeconfig
  # 创建密钥
  openssl genrsa -out kube-controller-manager.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=system:kube-controller-manager/emailAddress=admin@admin.com" \
  -key kube-controller-manager.key -out kube-controller-manager.csr

  # 使用kubernetes-ca进行签名
  openssl x509 -in kube-controller-manager.csr -out kube-controller-manager.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile ../kubernetes-ca.conf -extensions req_client

  rm -f kube-controller-manager.csr
  chmod 400 kube-controller-manager.key
  chmod 444 kube-controller-manager.crt

  # 把证书直接写道配置文件中，客户端不用保留证书了，(`--embed-certs`控制)
  cd ~/ca/kube-ca/kubeconfig
  export KUBECONFIG=controller-manager.conf
  kubectl config set-cluster default-cluster --server=https://localhost:6443 --certificate-authority ../kube-ca.crt --embed-certs
  kubectl config set-credentials default-controller-manager --client-key kube-controller-manager.key --client-certificate kube-controller-manager.crt --embed-certs
  kubectl config set-context default-system --cluster default-cluster --user default-controller-manager
  kubectl config use-context default-system
  ```

* scheduler.conf

  注意：证书的CN不要变更

  ```sh
  cd ~/ca/kube-ca/kubeconfig
  # 创建密钥
  openssl genrsa -out kube-scheduler.key 2048

  # 创建证书请求文件
  openssl req -new -sha256 \
  -subj "/C=CN/ST=ShangHai/L=ShangHai/O=net-cc.com/OU=net-cc.com/CN=system:kube-scheduler/emailAddress=admin@admin.com" \
  -key kube-scheduler.key -out kube-scheduler.csr

  # 使用kubernetes-ca进行签名
  openssl x509 -in kube-scheduler.csr -out kube-scheduler.crt \
  -CA ../kube-ca.crt -CAkey ../kube-ca.key -CAcreateserial \
  -req -sha256 -days 7300 \
  -extfile ../kubernetes-ca.conf -extensions req_client

  rm -f kube-scheduler.csr
  chmod 400 kube-scheduler.key
  chmod 444 kube-scheduler.crt

  # 把证书直接写道配置文件中，客户端不用保留证书了，(`--embed-certs`控制)
  export KUBECONFIG=scheduler.conf	
  kubectl config set-cluster default-cluster --server=https://localhost:6443 --certificate-authority ../kube-ca.crt --embed-certs
  kubectl config set-credentials kube-scheduler --client-key kube-scheduler.key --client-certificate kube-scheduler.crt --embed-certs
  kubectl config set-context default-system --cluster default-cluster --user kube-scheduler
  kubectl config use-context default-system
  ```

## 复制证书到各节点

* server节点

  ```sh
  # 复制kube-apiserver证书/密钥到两个server节点
  cd ~/ca/kube-ca/apiserver
  cp ../kube-ca.crt apiserver.crt apiserver-kubelet-client.crt sa.pub /opt/kubernetes/ssl/
  cp ../kube-ca.key apiserver.key apiserver-kubelet-client.key sa.key /opt/kubernetes/ssl/
  scp ../kube-ca.crt apiserver.crt apiserver-kubelet-client.crt sa.pub root@node4:/opt/kubernetes/ssl/
  scp ../kube-ca.key apiserver.key apiserver-kubelet-client.key sa.key root@node4:/opt/kubernetes/ssl/

  # 复制front-proxy-client证书/密钥到两个server节点
  cd ~/ca/kube-proxy-ca/front-proxy-client
  cp front-proxy-client.key front-proxy-client.crt ../front-proxy-ca.crt /opt/kubernetes/ssl/
  scp front-proxy-client.key front-proxy-client.crt ../front-proxy-ca.crt root@node4:/opt/kubernetes/ssl/

  # 复制controller-manager、scheduler连接配置文件到所有server节点
  cd ~/ca/kube-ca/kubeconfig
  mkdir ~/.kube/
  unset KUBECONFIG # 恢复kubectl的环境变量
  cp admin.conf ~/.kube/config
  cp controller-manager.conf scheduler.conf	 /opt/kubernetes/etc/
  scp controller-manager.conf scheduler.conf root@node4:/opt/kubernetes/etc/
  ```

* node节点

  ```sh
  # 复制kubelet配置文件到所有node节点
  cd ~/ca/kube-ca/kubeconfig
  scp kubelet-node1.conf proxy.conf root@node1:/opt/kubernetes/etc/
  # 复制kubelet的服务证书
  cd ~/ca/kube-ca/kubelet
  scp ../kube-ca.crt  kubelet.crt kubelet.key root@node1:/opt/kubernetes/ssl/
  scp ../kube-ca.crt  kubelet.crt kubelet.key root@node2:/opt/kubernetes/ssl/

  ```

* node节点现有文件

  ```sh
  tree /opt/kubernetes
  /opt/kubernetes
  ├── bin
  ├── data
  ├── etc
  │   └── kubelet-node1.conf  # node1才有，node2没有
  │   └── proxy.conf
  |   └── conf.d
  ├── logs
  └── ssl
  │   └── kube-ca.crt
  │   └── kubelet.crt
  │   └── kubelet.key
  ```

* Master节点现有文件

  ```sh
  $ tree /opt/kubernetes
  /opt/kubernetes
  ├── bin
  ├── data
  ├── etc
  │   ├── controller-manager.conf
  │   └── scheduler.conf	
  ├── logs
  └── ssl
      ├── apiserver.crt
      ├── apiserver.key
      ├── apiserver-kubelet-client.crt
      ├── apiserver-kubelet-client.key
      ├── etcd-ca.crt
      ├── etcd-client.crt
      ├── etcd-client.key
      ├── front-proxy-client.crt
      ├── front-proxy-client.key
      ├── front-proxy-ca.crt
      ├── kube-ca.key
      ├── kube-ca.crt
      ├── sa.key
      └── sa.pub
  ```

# Master Node(主)

下边是一些公共参数，主要是日志配置，节点上的所有程序都调用了

```sh
# 公共配置文件Controller Manager、scheduler都调用了
cat > /opt/kubernetes/etc/kube-default.conf<<EOF
# k8s所有服务共用配置文件
KUBE_LOG_ARGS="--logtostderr=false --log-dir=/opt/kubernetes/logs/ --v=2"
EOF
```

* `--logtostderr`: 设置为false表示将日志写入文件，不写入stderr

* `--log-dir`: 日志目录

* `--v`: 日志等级


**先安装一个主Master节点，另一个等单节点集群创建成功后在添加**

## 下载程序

[此链接查看最新版](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)

```sh
# 下载server端程序
cd /tmp
wget https://dl.k8s.io/v1.19.1/kubernetes-server-linux-amd64.tar.gz
tar -zxf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin/
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin/
# 传到另一个master节点 省的在下载了
scp kube-apiserver kube-scheduler kube-controller-manager root@node4:/opt/kubernetes/bin/
cd /tmp
rm -rf kubernetes*
```

## 部署API Server

主节点上负责提供 Kubernetes API 服务的组件；它是 Kubernetes 控制面的前端。

### systemd服务

```sh
# 添加api-server到systemd服务
cat > /usr/lib/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/opt/kubernetes/etc/kube-apiserver.conf
EnvironmentFile=/opt/kubernetes/etc/kube-default.conf
User=kube
Group=kube
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_API_ARGS \$KUBE_LOG_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 配置文件

```sh
# api-server配置文件,
cat > /opt/kubernetes/etc/kube-apiserver.conf<<EOF
KUBE_API_ARGS="--storage-backend=etcd3 \\
--etcd-servers=https://node1:2379,https://node3:2379,https://node2:2379 \\
--etcd-cafile=/opt/kubernetes/ssl/etcd-ca.crt \\
--etcd-certfile=/opt/kubernetes/ssl/etcd-client.crt \\
--etcd-keyfile=/opt/kubernetes/ssl/etcd-client.key \\
--bind-address=0.0.0.0 \\
--advertise-address=0.0.0.0 \\
--secure-port=6443 \\
--insecure-port=0 \\
--service-cluster-ip-range=10.110.0.0/16 \\
--service-node-port-range=30000-32767 \\
--allow-privileged=true \\
--authorization-mode=RBAC,Node \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log \\
--enable-admission-plugins=NodeRestriction,PodPreset \\
--runtime-config=settings.k8s.io/v1alpha1=true \\
--client-ca-file=/opt/kubernetes/ssl/kube-ca.crt \\
--kubelet-client-certificate=/opt/kubernetes/ssl/apiserver-kubelet-client.crt \\
--kubelet-client-key=/opt/kubernetes/ssl/apiserver-kubelet-client.key \\
--tls-cert-file=/opt/kubernetes/ssl/apiserver.crt  \\
--tls-private-key-file=/opt/kubernetes/ssl/apiserver.key \\
--service-account-key-file=/opt/kubernetes/ssl/sa.pub \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/etc/token.csv \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.crt \\
--proxy-client-key-file=/opt/kubernetes/ssl/front-proxy-client.key \\
--proxy-client-cert-file=/opt/kubernetes/ssl/front-proxy-client.crt \\
--requestheader-allowed-names=aggregator,front-proxy-client \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \\
--enable-aggregator-routing=true"
EOF

# 创建token.csv，记住token，等会添加第二个节点需要用到
# 格式：token，用户名，UID，用户组
TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > /opt/kubernetes/etc/token.csv << EOF
$TOKEN,kubelet-bootstrap,10001,"system:bootstrappers"
EOF
```

配置文件说明：

* `--storage-backend`: 指定etcd的版本

* `--etcd-servers`: etcd服务的URL

* `--etcd-*file`: 连接etcd的证书和密钥

* `--advertise-address`: 集群通告地址

* `--secure-port`: API Server绑定主机安全端口号，默认为6443

* `--service-cluster-ip-range`: Kubernetes集群中Service的虚拟IP地址范围，该IP范围不能与物理机的IP地址有重合。

* `--service-node-port-range`: Kubernetes集群中Service可使用的物理机端口号范围，默认值为30000～32767

* `--allow-privileged`: 启用授权

* `authorization-mode`: 认证授权，启用RBAC授权和节点自管理

* `audit-log*`: 审计日志

* `--enable-admission-plugins`: Kubernetes集群的准入控制设置，各控制模块以插件的形式依次生效。默认启用的有(NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority, DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize, RuntimeClass, CertificateApproval, CertificateSigning, CertificateSubjectRestriction, DefaultIngressClass, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ResourceQuota)

* `--runtime-config`: 传递给 apiserver 用于描述运行时配置的键值对集合。 apis/<groupVersion> 键可以被用来打开/关闭特定的 api 版本。apis/<groupVersion>/<resource> 键被用来打开/关闭特定的资源 . api/all 和 api/legacy 键分别用于控制所有的和遗留的 api 版本

* `--client-ca-file`: 对于任何请求，如果存包含client-ca-file中的authorities签名的客户端证书，将会使用客户端证书中的 CommonName 对应的身份进行认证

* `--kubelet-client-*`: apiserver访问kubelet客户端证书

* `--tls-*-file`: apiserver https证书

* `--service-account-key-file`: 服务帐户密钥对

* `--enable-bootstrap-token-auth`: 允许 'kube-system' 命名空间中的 'bootstrap.kubernetes.io/token' 类型密钥可以被用于 TLS 的启动认证

* `--token-auth-file`: 这个文件将被用于通过令牌认证来保护 API 服务的安全端口

备注：[apiserver所有参数的官方说明连接](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)


## 部署Controller Manager

在主节点上运行控制器的组件。

从逻辑上讲，每个控制器都是一个单独的进程，但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。

这些控制器包括:

* 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应。
* 副本控制器（Replication Controller）: 负责为系统中的每个副本控制器对象维护正确数量的 Pod。
* 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)。
* 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌.

### systemd服务

```sh
# 添加controller-manager到systemd服务
cat > /usr/lib/systemd/system/kube-controller-manager.service<<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Wants=kube-apiserver.service

[Service]
EnvironmentFile=/opt/kubernetes/etc/kube-controller-manager.conf
EnvironmentFile=/opt/kubernetes/etc/kube-default.conf
User=kube
Group=kube
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_ARGS \$KUBE_LOG_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 配置文件

```sh
cat > /opt/kubernetes/etc/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_ARGS="--leader-elect=true \\
--kubeconfig=/opt/kubernetes/etc/controller-manager.conf \\
--authentication-kubeconfig=/opt/kubernetes/etc/controller-manager.conf \\
--authorization-kubeconfig=/opt/kubernetes/etc/controller-manager.conf \\
--bind-address=0.0.0.0 \\
--port=0 \\
--cert-dir=/opt/kubernetes/ssl/ \\
--allocate-node-cidrs=true \\
--configure-cloud-routes=false \\
--cloud-provider=external \\
--cluster-cidr=10.120.0.0/16 \\
--service-cluster-ip-range=10.110.0.0/16 \\
--client-ca-file=/opt/kubernetes/ssl/kube-ca.crt \\
--root-ca-file=/opt/kubernetes/ssl/kube-ca.crt \\
--controllers=*,bootstrapsigner,tokencleaner,-cloud-node-lifecycle \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/kube-ca.crt \\
--cluster-signing-key-file=/opt/kubernetes/ssl/kube-ca.key \\
--cluster-signing-duration=87600h0m0s \\
--service-account-private-key-file=/opt/kubernetes/ssl/sa.key \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.crt \\
--requestheader-allowed-names=aggregator,front-proxy-client \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--use-service-account-credentials=true"
EOF
```

配置文件说明：

* `--leader-elect`：当该组件启动多个时，自动选举（HA）

* `--kubeconfig`: 指向 kubeconfig 文件的路径。该文件中包含主控节点位置以及鉴权凭据信息。	

* `--authentication-kubeconfig`: kubeconfig 文件的路径名。该文件中包含与某 Kubernetes “核心” 服务器相关的信息，并支持足够的权限以创建 tokenreviews.authentication.k8s.io。此选项是可选的。如果设置为空值，所有令牌请求都会被认作匿名请求，Kubernetes 也不再在集群中查找客户端的 CA 证书信息。

* `--authorization-kubeconfig`: 包含 Kubernetes “核心” 服务器信息的 kubeconfig 文件路径，所包含信息具有创建 subjectaccessreviews.authorization.k8s.io 的足够权限。此参数是可选的。如果配置为空字符串，未被鉴权模块所忽略的请求都会被禁止

* `--cert-dir`: TLS 证书所在的目录。如果提供了--tls-cert-file 和 --tls private-key-file，则将忽略此参数

* `--allocate-node-cidrs`: 为 Pod 分配和设置子网掩码

* `--configure-cloud-routes`: 决定是否由 --allocate-node-cidrs 所分配的 CIDR 要通过云驱动程序来配置。	

* `--cloud-provider`: 云服务的提供者。`external`表示没有对应的提供者（驱动）

* `--cluster-cidr`: Kubernetes集群中pod的IP地址范围
  
* `--service-cluster-ip-range`: Kubernetes集群中Service的虚拟IP地址范围，该IP范围不能与物理机的IP地址有重合。

* `--client-ca-file`: 对于所有能够提供客户端证书的请求，若该证书由 client-ca-file 中所给机构之一签署，则该请求会被成功认证为客户端证书中 CommonName 所给的实体

* `--root-ca-file`: 服务账号的令牌Secret中会包含此根证书机构

* `--controllers`：要启用的控制器列表。* 表示启用所有默认启用的控制器, 这里我禁用了云检测

* `--cluster-signing-cert-file/--cluster-signing-key-file`：自动为kubelet颁发证书的CA，与apiserver保持一致

* `--cluster-signing-duration`: 自动为kubelet颁发证书的有效期

* `--service-account-private-key-file`: 服务帐户密钥对
  
* `--use-service-account-credentials`: 为每个控制器单独使用服务账号凭据。

备注：[Controller Manager的所有参数说明连接](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-controller-manager/)

## 部署scheduler

主节点上的组件，该组件监视那些新创建的未指定运行节点的 Pod，并选择节点让 Pod 在上面运行。

### systemd服务

```sh
# 添加scheduler到systemd服务
cat > /usr/lib/systemd/system/kube-scheduler.service<<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Wants=kube-apiserver.service

[Service]
EnvironmentFile=/opt/kubernetes/etc/kube-scheduler.conf
EnvironmentFile=/opt/kubernetes/etc/kube-default.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_ARGS \$KUBE_LOG_ARGS
User=kube
Group=kube
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 配置文件

```sh
cat > /opt/kubernetes/etc/kube-scheduler.conf<<EOF
KUBE_SCHEDULER_ARGS="--leader-elect=true \\
--cert-dir=/opt/kubernetes/ssl/ \\
--config=/opt/kubernetes/etc/kube-scheduler.yaml \\
--authentication-kubeconfig=/opt/kubernetes/etc/scheduler.conf \\
--authorization-kubeconfig=/opt/kubernetes/etc/scheduler.conf	"
EOF

```

配置文件说明：

* `--leader-elect`：当该组件启动多个时，自动选举（HA）

* `--cert-dir`: TLS 证书所在的目录。如果提供了--tls-cert-file 和 --tls private-key-file，则将忽略此参数

* `--config`: 指配置文件的路径。命令行参数会覆盖此文件中的值。

* `--authentication-kubeconfig`: 指向具有足够权限以创建 subjectaccessreviews.authorization.k8s.io 的 'core' kubernetes 服务器的 kubeconfig 文件

备注: [所有scheduler的参数说明连接](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-scheduler/)

## 文件权限

```sh
mkdir -p /var/run/kubernetes /usr/libexec/kubernetes
echo 'd /var/run/kubernetes 0755 kube kube -' >> /usr/lib/tmpfiles.d/kubernetes.conf
chown kube:kube -R /var/run/kubernetes /usr/libexec/kubernetes /opt/kubernetes/{etc,logs,ssl,data}
restorecon -R /opt/kubernetes /usr/libexec/kubernetes /var/run/kubernetes
```

## 启动服务

```sh
systemctl start kube-apiserver.service
systemctl start kube-controller-manager.service
systemctl enable kube-apiserver.service
systemctl enable kube-controller-manager.service
```

## 生成scheduler配置文件

需要先启动API server

```sh
# 生成kube-scheduler.yaml默认文件
/opt/kubernetes/bin/kube-scheduler \
--authentication-kubeconfig=/opt/kubernetes/etc/scheduler.conf \
--authorization-kubeconfig=/opt/kubernetes/etc/scheduler.conf \
--kubeconfig=/opt/kubernetes/etc/scheduler.conf \
--authentication-tolerate-lookup-failure=false \
--write-config-to /opt/kubernetes/etc/kube-scheduler.yaml

# 启动scheduler
systemctl start kube-scheduler.service
systemctl enable kube-scheduler.service
```

## 查看集群状态

由于我的ca管理和kubectl程序都是安装在node3上的，所以在node3上操作

```sh
# 复制kubectl的配置文件
mkdir ~/.kube/
cp ~/ca/kube-ca/kubeconfig/admin.conf ~/.kube/config
export KUBECONFIG="~/.kube/config"

kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}  
```

## 授权APIserver访问kubelet

--kubelet-client-*参数调用证书的CN是`system:masters`所以用户就是`system:masters`, `system:masters`用户默认绑定到了集群管理员了，不要轻易使用哦~~

```sh
kubectl create clusterrolebinding system:kubelet-api-admin \
--clusterrole=system:kubelet-api-admin \
--user=system:masters
```

## 授权kubelet创建csr

刚刚生成的toke文件中的用户组是`system:bootstrappers`，
把用户组和RBAC预定义规则`system:node-bootstrapper`进行绑定既可以授权kubelet通过apiserver管理证书了

```sh
kubectl create clusterrolebinding system:kubelet-bootstrap \
--clusterrole system:node-bootstrapper \
--group system:bootstrappers
```

# Worker Node

注意：一个节点（node1）的kubelet使用手动生成的密钥/证书, 一个（node2）上kubelet的密钥证书是由TLS Bootstrapping管理的，所以要两个节点上的kubelet配置是不一样的，其他都一样。

下边是一些公共参数，主要是日志配置，节点上的所有程序都调用了

```sh
cat > /opt/kubernetes/etc/kube-default.conf<<EOF
# k8s所有服务共用配置文件
KUBE_LOG_ARGS="--logtostderr=false --log-dir=/opt/kubernetes/logs/ --v=2"
EOF
```

* `--logtostderr`: 设置为false表示将日志写入文件，不写入stderr

* `--log-dir`: 日志目录

* `--v`: 日志等级


## 安装docker

Kubernetes 支持多个容器运行环境: Docker、 containerd、cri-o、 rktlet 以及任何实现 Kubernetes CRI (容器运行环境接口),我才用docker，毕竟常用

参考我之前写的一篇文章来[安装docker]({{< relref "docker install.md" >}})

两个worker节点上安装配置都一样

注意docker的配置文件如下：

```sh
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
"registry-mirrors":["https://t88fp3u4.mirror.aliyuncs.com"],
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
    "max-size": "100m"
},
"iptables": false,
"ip-forward": false,
"ip-masq": false,
"userland-proxy": false,
"selinux-enabled": true,
"storage-driver": "overlay2",
"storage-opts": [
    "overlay2.override_kernel_check=true"
]
}
EOF
```

## 下载程序文件

[此链接查看最新版](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)

```sh
cd /tmp
wget https://dl.k8s.io/v1.19.1/kubernetes-node-linux-amd64.tar.gz
tar -zxf kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin/kubelet
sudo cp kubelet kube-proxy /opt/kubernetes/bin/
# 传到另一各节点上，以免再次下载
scp kubelet kube-proxy root@node2:/opt/kubernetes/bin/
cd /tmp
rm -rf kubernetes*
```

## 部署kubelet(node1)

一个在集群中每个节点上运行的代理。它保证容器都运行在 Pod 中。

kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。kubelet 不会管理不是由 Kubernetes 创建的容器。

这里我先写在**node1节点上通过手动管理证书的方式来启动kubelet**，另一个等会在说

### systemd服务

```sh
# 设置kubelet的systemd服务
cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Wants=docker.service

[Service]
WorkingDirectory=/opt/kubernetes/data/
EnvironmentFile=/opt/kubernetes/etc/kube-kubelet.conf
EnvironmentFile=/opt/kubernetes/etc/kube-default.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBE_LOG_ARGS \$KUBE_KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### 配置文件

```sh
# 注意--kubeconfig的参数，文件调用本机的文件
cat > /opt/kubernetes/etc/kube-kubelet.conf<<EOF
KUBE_KUBELET_ARGS="\\
--kubeconfig=/opt/kubernetes/etc/kubelet-$HOSTNAME.conf \\
--hostname-override=$HOSTNAME \\
--network-plugin=cni \\
--config=/opt/kubernetes/etc/kube-kubelet.yaml \\
--dynamic-config-dir=/opt/kubernetes/etc/conf.d \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=951021/pause"
EOF

# 注意cgroupDriver修改容器的cgroup一样，默认为：cgroupfs
cat > /opt/kubernetes/etc/kube-kubelet.yaml<<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
tlsCertFile: /opt/kubernetes/ssl/kubelet.crt
tlsPrivateKeyFile: /opt/kubernetes/ssl/kubelet.key
clusterDNS:
- 10.110.0.2
clusterDomain: cluster.local
authentication:
  x509:
    clientCAFile: /opt/kubernetes/ssl/kube-ca.crt
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
staticPodPath: /opt/kubernetes/data/manifests
failSwapOn: false
hairpinMode: hairpin-veth
EOF
```

配置文件说明：

* `--kubeconfig`:该文件将用于获取 kubelet 的客户端证书。如果 --kubeconfig 指定的文件不存在，则使用引导 kubeconfig 从 API 服务器请求客户端证书。成功后，将引用生成的客户端证书和密钥的 kubeconfig 文件写入 --kubeconfig 所指定的路径。客户端证书和密钥文件将存储在 --cert-dir 指向的目录中。

* `--hostname-override`: 设置本Node的名称。

* `--network-plugin`: 启用CNI

* `--config`: kubelet 将从该文件加载其初始配置。该路径可以是绝对路径，也可以是相对路径。相对路径从 kubelet 的当前工作目录开始。省略此参数则使用内置的默认配置值。命令行参数会覆盖此文件中的配置。

* `--cert-dir`: LS 证书所在的目录。如果设置了 --tls-cert-file 和 --tls-private-key-file，则该设置将被忽略。（默认值为 “/var/lib/kubelet/pki”）

* `--pod-infra-container-image`: 指定基础设施镜像，Pod 内所有容器与其共享网络和 IPC 命名空间。仅当容器运行环境设置为 docker 时，此特定于 docker 的参数才有效。（默认值为 “k8s.gcr.io/pause:3.1”）默认的由于某种不可描述的原因，无法访问，自己想办法吧。

备注：[所有kubelet参数说明连接](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/)

### 安装一些需要用到的工具

```sh
yum install conntrack-tools libnetfilter_cthelper libnetfilter_cttimeout libnetfilter_queue socat-y
```

## 部署kubelet(node2)

**这个节点的kubelet的密钥证书是由TLS Bootstrapping管理的**

### systemd服务

```sh
# 设置kubelet的systemd服务
cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Wants=docker.service

[Service]
WorkingDirectory=/opt/kubernetes/data/
EnvironmentFile=/opt/kubernetes/etc/kube-kubelet.conf
EnvironmentFile=/opt/kubernetes/etc/kube-default.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBE_LOG_ARGS \$KUBE_KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### 配置文件

```sh
# 注意--kubeconfig的参数，文件调用本机的文件
cat > /opt/kubernetes/etc/kube-kubelet.conf<<EOF
KUBE_KUBELET_ARGS="\\
--kubeconfig=/opt/kubernetes/etc/kubelet-$HOSTNAME.conf \\
--bootstrap-kubeconfig=/opt/kubernetes/etc/bootstrap.conf \\
--hostname-override=$HOSTNAME \\
--dynamic-config-dir=/opt/kubernetes/etc/conf.d \\
--network-plugin=cni \\
--config=/opt/kubernetes/etc/kube-kubelet.yaml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-shanghai.aliyuncs.com/net-cc/pause"
EOF

# 注意cgroupDriver修改容器的cgroup一样，默认为：cgroupfs
cat > /opt/kubernetes/etc/kube-kubelet.yaml<<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
tlsCertFile: /opt/kubernetes/ssl/kubelet.crt
tlsPrivateKeyFile: /opt/kubernetes/ssl/kubelet.key
clusterDNS:
- 10.110.0.2
clusterDomain: cluster.local
authentication:
  x509:
    clientCAFile: /opt/kubernetes/ssl/kube-ca.crt
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
failSwapOn: false
hairpinMode: hairpin-veth
EOF
```


* 这里新增加了一个参数`--bootstrap-kubeconfig`用来首次启动向apiserver申请证书

* `bootstrap-kubeconfig`文件生成方法：

  由于我的kubectl安装在master 节点(node3)上的，所以需要在master节点上生成，然后复制到worker节点的`/opt/kubernetes/etc/`目录

  ```sh
  cd ~/ca/kube-ca/kubeconfig
  KUBE_APISERVER="https://node3:6443" # apiserver IP:PORT
  TOKEN="c47ffb939f5ca36231d9e3121a252940" # 与token.csv里保持一致
  # 生成bootstrap配置文件
  kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/kube-ca.crt --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=bootstrap.conf
  kubectl config set-credentials system:bootstrappers --token=${TOKEN} --kubeconfig=bootstrap.conf
  kubectl config set-context default --cluster=kubernetes --user=system:bootstrappers --kubeconfig=bootstrap.conf
  kubectl config use-context default --kubeconfig=bootstrap.conf
  # 传到node2节点
  scp bootstrap.conf root@node2:/opt/kubernetes/etc/bootstrap.conf
  ```

### 启动服务

```sh
restorecon -R /opt/kubernetes
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```

### 批准kubelet证书申请并加入集群

```sh
# 查看kubelet证书请求
kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A   6m3s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 批准申请
kubectl certificate approve node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A

# 查看节点
kubectl get node
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   <none>   7s    v1.18.3
```

注：由于网络插件还没有部署，节点会没有准备就绪 NotReady

### 安装一些需要用到的工具

```sh
yum install conntrack-tools -y
```


**注意**：这里的`--kubeconfig`指向的配置文件是空的，等会会自动生成

## 部署kube-proxy

kube-proxy 是集群中每个节点上运行的网络代理,实现 Kubernetes Service 概念的一部分。

kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。

如果操作系统提供了数据包过滤层并可用的话，kube-proxy会通过它来实现网络规则。否则，kube-proxy 仅转发流量本身。

两个worker节点上安装配置都一样

### systemd服务

```sh
# 设置kube-proxy的systemd服务
cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/opt/kubernetes/data/
EnvironmentFile=/opt/kubernetes/etc/kube-proxy.conf
EnvironmentFile=/opt/kubernetes/etc/kube-default.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_ARGS \$KUBE_LOG_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### 配置文件

```sh
cat > /opt/kubernetes/etc/kube-proxy.conf<<EOF
KUBE_PROXY_ARGS="--config=/opt/kubernetes/etc/kube-proxy.yaml"
EOF

# 生成配置文件
/opt/kubernetes/bin/kube-proxy \
--kubeconfig=/opt/kubernetes/etc/proxy.conf \
--write-config-to=/opt/kubernetes/etc/kube-proxy.yaml

# 修改配置文件的以下几个地方
hostnameOverride: $HOSTNAME
.........
clusterCIDR: =10.120.0.0/16
........
ipvs:
........
  scheduler: "rr"
........
mode: "ipvs"
```

配置文件说明：

* `--config`: 配置文件的路径。
* `--kubeconfig`: 包含授权信息的 kubeconfig 文件的路径（master 位置由 master 标志设置）
* `--write-config-to`: 将配置值写入此文件并退出程序

注意：[所有配置参数说明连接](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/)


## 文件权限

```sh
restorecon -R /opt/kubernetes
```

## 启动服务

```sh
sudo systemctl daemon-reload
sudo systemctl start kubelet.service
sudo systemctl start kube-proxy.service
sudo systemctl enable kubelet.service
sudo systemctl enable kube-proxy.service
```

# 部署CNI网络

通过给Kubelet传递--network-plugin=cni命令行选项来选择CNI插件。Kubelet从--cni-conf-dir（默认是/etc/cni/net.d）读取文件并使用该文件中的CNI配置来设置每个pod的网络。CNI配置文件必须与CNI规约匹配，并且配置引用的任何所需的CNI插件都必须存在于--cni-bin-dir（默认是/opt/cni/bin）。

如果这个目录中有多个CNI配置文件，kubelet将会使用按文件名的字典顺序排列的第一个作为配置文件。

除了配置文件指定的CNI插件外，Kubernetes还需要标准的CNIlo插件，最低版本是0.2.0。

[查看最新版本](https://github.com/containernetworking/plugins/releases)

* 安装CNI二进制文件

  ```sh
  # 两个Worker Node上都需要安装
  mkdir -p /opt/cni/bin/
  VERSION=v0.8.7
  cd /tmp
  curl -L https://github.com/containernetworking/plugins/releases/download/${VERSION}/cni-plugins-linux-amd64-${VERSION}.tgz -o cni-plugins-linux-amd64-${VERSION}.tgz
  tar -zxvf cni-plugins-linux-amd64-${VERSION}.tgz -C /opt/cni/bin/
  scp -r /opt/cni/bin/ root@node2:/opt/cni/bin/
  rm -f cni-plugins-linux-amd64-${VERSION}.tgz
  ```

* 安装CNI网络插件`flannel`

  flannel我部署到kube集群里边了，没有单独每个Worker Node部署

  ```sh
  # 在master节点操作，因为master节点安装了kubectl客户端程序和配置
  wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  PODIP=10.120.0.0/16  # pod的ip地址，即controller-manager控制器设置的cluster-cidr地址
  sed -i 's|10.244.0.0/16|'"$PODIP"'|g' kube-flannel.yml
  kubectl apply -f kube-flannel.yml
  ```

# Master Node(备)

和Master Node主配置一样，不在叙述

# 负载均衡


## 部署nginx

* 安装nginx

  在两个master node上安装

  ```sh
  yum install -y nginx
  ```

* 配置nginx

  使用nginx的4层负载均衡，主备一样

  ```sh
  setsebool -P nis_enabled 1 # 修改selinux权限
  cp /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
  cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
  cat >> /etc/nginx/nginx.conf <<EOF
  # 四层负载均衡，为两台Master apiserver组件提供负载均衡
  stream {

      log_format  main  '\$remote_addr \$upstream_addr - [\$time_local] \$status \$upstream_bytes_sent';
      access_log  /var/log/nginx/k8s.access.log  main;
      upstream k8s-apiserver {
        server 192.168.122.12:6443;   # Master1 APISERVER IP:PORT
        server 192.168.122.13:6443;   # Master2 APISERVER IP:PORT
      }

      server {
        listen 8443;
        proxy_pass k8s-apiserver;
      }
  }
  EOF
  ```

* 启动nginx

  ```sh
  systemctl start nginx
  systemctl enable nginx
  ```

## 部署keepalived

* 安装keepalived
  
  在两个master node上安装keepalived即可（一主一备）

  ```sh
  yum install -y keepalived
  mkdir -p /etc/keepalived
  ```

* master节点1(node3)配置文件

  ```sh
  mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.back
  cat > /etc/keepalived/keepalived.conf<<EOF
  ! Configuration File for keepalived
  global_defs {
    router_id keepalive-master
  }

  vrrp_script check_apiserver {
    # 检测脚本路径
    script "/etc/keepalived/check-nginx.sh"
    # 多少秒检测一次
    interval 3
    # 失败的话权重-2
    weight -2
    fail 1
  }

  vrrp_instance VI-kube-master {
    state MASTER    # 定义节点角色
    interface ens3  # 网卡名称
    virtual_router_id 68
    priority 100
    # 启用单播模式~
    unicast_src_ip 192.168.122.12
    unicast_peer {
      192.168.122.13
    }
    dont_track_primary
    advert_int 3
    authentication {
        auth_type PASS
        auth_pass net-cc
    }
    virtual_ipaddress {
      # 自定义虚拟ip
      192.168.122.2
    }
    track_script {
        check_apiserver
    }
  }
  EOF
  ```

* master节点2(node4)配置文件

  ```sh
  mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.back
  cat > /etc/keepalived/keepalived.conf<<EOF
  ! Configuration File for keepalived
  global_defs {
    router_id keepalive-backup
  }

  vrrp_script check_apiserver {
    script "/etc/keepalived/check-nginx.sh"
    interval 3
    weight -5
    fail 1
  }

  vrrp_instance VI-kube-master {
    state BACKUP
    interface ens32
    virtual_router_id 68
    priority 95
    unicast_src_ip 192.168.122.13
    unicast_peer {
      192.168.122.12
    }
    dont_track_primary
    advert_int 3
    authentication {
        auth_type PASS
        auth_pass net-cc
    }
    virtual_ipaddress {
      192.168.122.2
    }
    track_script {
        check_apiserver
    }
  }
  EOF
  ```

* 两个节点都创建以下脚本文件

  ```sh
  cat > /etc/keepalived/check-nginx.sh<<EOF
  #!/bin/bash
  #检查ngin进程是否存在
  counter=\$(ps -C nginx --no-heading|wc -l)
  if [ "\${counter}" = "0" ]; then
    #尝试启动一次ngnx,停止5秒后再次检测
    systemctl restart nginx
    sleep 5
    counter=\$(ps -C nginx --no-heading|wc -l)
    if [ "\${counter}"="0" ]; then
      #如果启动没成功,就杀掉keepave触发主备切换
      systemctl stop nginx
      exit 2
    fi
  fi
  EOF
  chmod +x /etc/keepalived/check-nginx.sh
  ```

* 启动keepalived

  ```sh
  systemctl enable keepalived && service keepalived start
  ```

* 测试
  
  ```sh
  curl -k https://192.168.122.2:6443/version
  {
  "major": "1",
  "minor": "19",
  "gitVersion": "v1.19.1",
  "gitCommit": "206bcadf021e76c27513500ca24182692aabd17e",
  "gitTreeState": "clean",
  "buildDate": "2020-09-09T11:18:22Z",
  "goVersion": "go1.15",
  "compiler": "gc",
  "platform": "linux/amd64"
  }
  ```

## 测试

* 关闭主节点Nginx，测试VIP是否漂移到备节点服务器。

  ```sh
  # 在Nginx Master执行 pkill nginx
  pkill nginx
  # 在Nginx Backup，ip addr命令查看已成功绑定VIP
  ip addr
  ```

* 访问负载均衡器测试

  找K8s集群中任意一个节点，使用curl查看K8s版本测试，使用VIP访问：

  ```sh
  curl -k https://192.168.122.2:6443/version
  {
    "major": "1",
    "minor": "18",
    "gitVersion": "v1.18.3",
    "gitCommit": "2e7996e3e2712684bc73f0dec0200d64eec7fe40",
    "gitTreeState": "clean",
    "buildDate": "2020-05-20T12:43:34Z",
    "goVersion": "go1.13.9",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
  ```

## 修改所有Worker Node连接LB VIP

虽然我们增加了Master2和负载均衡器，但是我们是从单Master架构扩容的，也就是说目前所有的Node组件连接都还是Master1，如果不改为连接VIP走负载均衡器，那么Master还是单点故障。
因此接下来就是要改所有Node组件配置文件

* 所有Worker Node执行：

  ```
  sed -i 's#192.168.122.12#k8s.api#g' /opt/kubernetes/etc/*
  sed -i 's#node3#k8s.api#g' /opt/kubernetes/etc/*
  systemctl restart kubelet
  systemctl restart kube-proxy
  ```

* 检查节点状态：

  ```sh
  kubectl get node
  NAME          STATUS   ROLES    AGE    VERSION
  k8s-master    Ready    <none>   34h    v1.18.3
  k8s-master2   Ready    <none>   101m   v1.18.3
  k8s-node1     Ready    <none>   33h    v1.18.3
  k8s-node2     Ready    <none>   33h    v1.18.3
  ```

至此，一套完整的 Kubernetes 高可用集群就部署完成了！

PS：如果你是在公有云上，一般都不支持keepalived，那么你可以直接用它们的负载均衡器产品（内网就行，还免费~），架构与上面一样，直接负载均衡多台Master kube-apiserver即可！