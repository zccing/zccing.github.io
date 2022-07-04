---
title: "使用kubeadm部署高可用k8s集群"
date: 2020-10-07T22:11:16+08:00
lastmod: 2020-12-05T22:07:25+08:00
draft: true
keywords: ['k8s', 'kubeadm安装k8s','k8s集群','容器']
description: "使用kubeadm安装k8s高可用集群"
tags: ['kubernetes']
categories: ['containers']
autoCollapseToc: true
---

使用kubeadm部署Kubernetes集群，我只使用一台nginx模拟高可用，所以实际环境最好上负载均衡
<!--more-->

# 节点信息

| IP地址        | 主机名称 | DNS记录          | 类型              | 系统    |
| ------------- | -------- | ---------------- | ----------------- | ------- |
| 192.168.122.1 | node1    | node1.net-cc.com | Worker Node/nginx | centos7 |
| 192.168.122.2 | node2    | node2.net-cc.com | Master Node/etcd  | centos7 |
| 192.168.122.3 | node3    | node3.net-cc.com | Master Node/etcd  | centos7 |
| 192.168.122.4 | node4    | node4.net-cc.com | Master Node/etcd  | centos7 |
| 192.168.122.4 | node4    | api.net-cc.com   | nginx             | centos7 |
| 10.110.0.0/16 |          |                  | k8s service       |         |
| 10.120.0.0/16 |          |                  | k8s pod           |         |

电脑配置较低，这么多虚拟机都要跑不动~~

## 初始化节点

```bash
# 关闭swap, 我没关，苦逼，内存不够
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久
# 根据规划设置主机名
hostnamectl set-hostname <hostname>
# 添加dns解析记录
cat >> /etc/hosts << EOF
192.168.122.1 node1.net-cc.com
192.168.122.2 node2.net-cc.com
192.168.122.3 node3.net-cc.com
192.168.122.4 node4.net-cc.com
192.168.122.4 api.net-cc.com
EOF

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

# 时间同步
sed -i '3i server ntp.aliyun.com iburst' /etc/chrony.conf
systemctl restart chronyd.service

# selinux设置为permissive，如果要开启selinux的时候，要先解决selinux拒绝日志记录
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
setsebool domain_kernel_load_modules 1

# 修改防火墙规则,我直接关闭了防火墙
systemctl stop firewalld
systemctl disable firewalld
firewall-cmd --permanent --add-port=6443/tcp # kube-api 安全端口
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=8472/udp
firewall-cmd --permanent --add-port=10250-10252/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# 加载内核模块
## ipvs模块
cat > /etc/modules-load.d/k8s.conf<<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
br_netfilter
nf_conntrack_ipv4
EOF
## 生效
systemctl restart systemd-modules-load.service

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system  # 生效
```

防火墙放行的端口介绍：

| 协议 | 方向 | 港口范围     | 目的                    | 使用                      |
| ---- | ---- | ------------ | ----------------------- | ------------------------- |
| TCP  | 入站 | 6443*，8080* | Kubernetes API server   | 所有                      |
| TCP  | 入站 | 2379-2380    | etcd server client API  | etcd                      |
| TCP  | 入站 | 10250        | Kubelet API             | 本机，控制平面组件        |
| TCP  | 入站 | 10251        | kube-scheduler          | 本机                      |
| TCP  | 入站 | 10252        | kube-controller-manager | 本机                      |
| TCP  | 入站 | 30000-32767  | NodePort Services**     | 所有node节点的service端口 |

**NodePort Services的默认端口范围。**

**使用 * 标记的端口号都可以自定义的，所以您需要保证所自定义的端口是开放的。**

# 部署docker

通过阿里云镜像仓库安装最新版docker，所有节点上都要安装

```bash
yum install -y yum-utils lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 注意： centos8需要更改为centos8的仓库，已经有centos8的镜像了
sed -i "s|centos/7|centos/8|g" /etc/yum.repos.d/docker-ce.repo
sudo yum makecache fast && yum install docker-ce && systemctl enable docker

# 配置文件
## 使用阿里云进行镜像加速
## cgroup模式使用systemd
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

# 启动docker 
systemctl enable docker
systemctl daemon-reload
systemctl start docker
```

# 部署IPVS

所有节点上都要安装

```bash
# 安装ipset,一般默认都有
yum install ipset ipvsadm -y
```

# 负载均衡

使用nginx模拟了一个负载均衡设备，要根据具体环境来搭建或使用云供应商的负载均衡

在worker节点上安装nginx, 实验环境，凑合下...

使用nginx的4层负载均衡，主备一样

```bash
cat > /etc/yum.repos.d/nginx.repo<<"EOF"
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
yum makecache fast
yum install nginx -y
setsebool -P nis_enabled 1 # 修改selinux权限

# 不开启80端口服务
mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
cat >> /etc/nginx/nginx.conf <<"EOF"
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log  /var/log/nginx/k8s.access.log  main;
    upstream k8s-apiserver {
      server node1.net-cc.com:6443;   # Master1 APISERVER IP:PORT
    # 后两个server先注释掉，因为我们不可能同时部署
    # server node2.net-cc.com:6443;   # Master2 APISERVER IP:PORT
    # server node3.net-cc.com:6443;   # Master2 APISERVER IP:PORT
    }

    server {
      listen 6443;
      proxy_pass k8s-apiserver;
    }
}
EOF

# 启动nginx
systemctl daemon-reload
systemctl enable nginx
systemctl start nginx
```

# 安装kubeadm kubelet

```bash
# 使用阿里云镜像站
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装
yum update -y && yum install -y kubelet kubeadm kubectl && systemctl enable --now kubelet
```

# kubectl命令补全

```bash
echo 'eval "$(kubectl completion bash)"' >> ~/.bashrc
echo 'eval "$(kubeadm completion bash)"' >> ~/.bashrc
source ~/.bashrc
```

# kubeadm init 配置文件
[完整配置](https://pkg.go.dev/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2#pkg-overview)

```bash
version="v1.19.4"
cluster_num=3
domain=".net-cc.com"
master_ip="192.168.123.1"
pod_network="10.120.0.0/16"
service_network="10.110.0.0/16"
dns_server="10.110.0.10"
control_plane_endpoint="api.net-cc.com"
image_repository="registry.aliyuncs.com/google_containers"
cat > kubeadm-init.conf <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
bootstrapTokens:
- token: "9a08jv.c0izixklcxtmnze7"
  description: "kubeadm bootstrap token"
  ttl: "24h"
- token: "783bde.3f89s0fje9f38fhf"
  description: "another bootstrap token"
  usages:
  - authentication
  - signing
  groups:
  - system:bootstrappers:kubeadm:default-node-token
localAPIEndpoint:
  advertiseAddress: $master_ip
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  # name: node1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  ignorePreflightErrors:
  - Swap
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
dns:
  type: CoreDNS
etcd:
  local:
    imageRepository: $image_repository
    dataDir: /var/lib/etcd
    extraArgs:
      debug: "false"
      enable-pprof: "true"
      logger: "zap"
      log-level: "warn"
    serverCertSANs:
    - etcd.net-cc.com
    peerCertSANs:
    - etcd.net-cc.com
  #  external:
  #   endpoints:
  #   - "10.100.0.1:2379"
  #   - "10.100.0.2:2379"
  #   caFile: "/etcd/kubernetes/pki/etcd/etcd-ca.crt"
  #   certFile: "/etcd/kubernetes/pki/etcd/etcd.crt"
  #   keyFile: "/etcd/kubernetes/pki/etcd/etcd.key"
imageRepository: $image_repository
kubernetesVersion: $version
networking:
  dnsDomain: cluster.local
  podSubnet: $pod_network
  serviceSubnet: $service_network
controlPlaneEndpoint: $control_plane_endpoint:6443
apiServer:
  timeoutForControlPlane: 4m0s
  extraArgs:
    v: "1"
    authorization-mode: Node,RBAC
    allow-privileged: "true"
    enable-admission-plugins: NodeRestriction,PodPreset
    runtime-config: settings.k8s.io/v1alpha1=true
    apiserver-count: $cluster_num
  extraVolumes:
  - name: "host-time"
    hostPath: "/etc/localtime"
    mountPath: "/etc/localtime"
    readOnly: true
    pathType: File
  certSANs:
  - $control_plane_endpoint
controllerManager: 
  extraArgs:
    v: "1"
    node-cidr-mask-size: "24"
    configure-cloud-routes: "false"
    cloud-provider: external
    bind-address: 0.0.0.0
    kube-api-burst: 400
    kube-api-qps: 200
    controllers: "*,bootstrapsigner,tokencleaner,-cloud-node-lifecycle"
  extraVolumes:
  - name: "host-time"
    hostPath: "/etc/localtime"
    mountPath: "/etc/localtime"
    readOnly: true
    pathType: File
scheduler: 
  extraArgs:
    v: "1"
    bind-address: 0.0.0.0
    authentication-tolerate-lookup-failure: "false"
  extraVolumes:
  - name: "host-time"
    hostPath: "/etc/localtime"
    mountPath: "/etc/localtime"
    readOnly: true
    pathType: File
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
bindAddressHardFail: false
clientConnection:
  acceptContentTypes: ""
  burst: 0
  # contentType: ""
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
  qps: 0
clusterCIDR: $pod_network
configSyncPeriod: 0s
conntrack:
  maxPerCore: null
  min: null
  tcpCloseWaitTimeout: null
  tcpEstablishedTimeout: null
detectLocalMode: ""
enableProfiling: false
healthzBindAddress: ""
hostnameOverride: ""
iptables:
  masqueradeAll: true
  masqueradeBit: null
  minSyncPeriod: 0s
  syncPeriod: 0s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: "lc"
  strictARP: false
  syncPeriod: 0s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
metricsBindAddress: "0.0.0.0"
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: null
portRange: "30000-32767"
showHiddenMetricsForVersion: ""
udpIdleTimeout: 0s
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- $dns_server
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
staticPodPath: /etc/kubernetes/manifests
failSwapOn: false
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
EOF
```

# 安装master节点
```bash
# 安装第一个master节点
systemctl enable kubelet
kubeadm init --config=kubeadm-init.conf --upload-certs
# 根据之前提示添加一个master节点,命令大概长这样

# node节点添加就不做介绍了，
```

# 安装flannel网络

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
PODIP=10.120.0.0/16  # pod的ip地址，即controller-manager控制器设置的cluster-cidr地址
sed -i 's|10.244.0.0/16|'"$PODIP"'|g' kube-flannel.yml
# 修改yml文件，修改时区为本机时区
apiVersion: apps/v1
kind: DaemonSet
......
spec:
......
  template:
  .......
    spec:
    .......
      containers:
      .........
        volumeMounts:
        - name: host-time
          mountPath: /etc/localtime
          readOnly: true
        .......
      volumes:
      - name: host-time
        hostPath:
          path: /etc/localtime
          type: File
      .........
# 集群安装flannel
kubectl apply -f kube-flannel.yml
```

# 安装

```bash
# 安装metrics-server(基础指标监控服务器)
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl apple -f components.yaml

# 安装dashboard
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
kubectl apple -f recommended.yaml
# 创建token
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# 查看token
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```


# 问题记录

```bash
# 错误日志,
root@node1:~# kubectl -n kube-system logs kube-proxy-lff8f | tail -n 10
I1118 21:46:43.893050       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:46:43.893229       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:47:21.012542       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:47:21.014367       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:47:35.082868       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:47:35.084358       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:48:20.363060       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:48:20.377840       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:48:38.525952       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
I1118 21:48:38.530152       1 streamwatcher.go:114] Unexpected EOF during watch stream event decoding: unexpected EOF
# 大概是1m中有一次异常中断，应该是ha的持久连接问题，修改haproxy的`timeout server` 和`timeout client`参数为10m后恢复


I1118 23:01:30.155050       1 client.go:360] parsed scheme: "passthrough"
I1118 23:01:30.155210       1 passthrough.go:48] ccResolverWrapper: sending update to cc: {[{https://192.168.123.1:2379  <nil> 0 <nil>}] <nil> <nil>}
I1118 23:01:30.155248       1 clientconn.go:948] ClientConn switching balancer to "pick_first"
```