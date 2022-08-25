---
title: "Routing information protocol(RIP)路由信息协议"
subtitle: "RIP"
date: 2022-08-23T21:05:57+08:00
lastmod: 2022-08-23T21:05:57+08:00
draft: false
author: "Cc"
authorLink: "www.net-cc.com"
description: "Routing information protocol(RIP)是一种基于距离矢量（Distance-Vector，D-V）算法的协议"

tags: ['rip']
categories: ['Route Protocol']

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

Routing information protocol(RIP)是一种基于距离矢量（Distance-Vector，D-V）算法的路由协议, 配置非常简单, RIP有Version 1和Version 2两个版本，Version 1已过时，所以本篇文章只记录Version 2的设置。
<!--more-->

## 1 基本概念

* RIP是一种基于距离矢量（Distance-Vector，D-V）算法的协议，它通过UDP报文进行路由信息的交换，使用的端口号为`520/UDP`。
* RIP使用跳数（Hop Count）来衡量到达目的地址的距离，称为度量值（Routing Cost）。
* 在 RIP 中，路由器到与它直接相连网络的跳数为 0，通过一个路由器可达的网络的跳数为 1，其余依此类推。
* 为限制收敛时间，RIP 规定度量值取 0～15 之 间的整数，大于或等于 16的跳数被定义为无穷大，即目的网络或主机不可达。
* 由于跳数限制，使得RIP不适合应用于大型网络。 
* 为提高性能，防止产生路由环路，RIP支持水平分割（Split Horizon）和毒性逆转（Poison Reverse）功能功能。
* RIP还可引入其它路由协议所得到的路由。

### 1.1 RIP的启动和运行过程

![RIP 工作过程](/images/network/rip/RIP工作过程.png "RIP 工作过程分析")

* `初始状态`：某路由器刚启动 RIP 时，以广播、组播或单播的形式向运行 RIP 协议的相邻路由器发送请求报文，相邻路由器收到请求报文后，响应该请求，回送包含本地路由表信息的响应报文。
* `构建路由表`：路由器收到响应报文后，更新本地路由表，同时向运行 RIP协议的相邻路由器发送触发更新报文，通告路由更新信息。相邻路由器收到触发更新报文后，又向其各自的相邻路由器发送触发更新报文。在一连串的触发更新后，各路由器都能得到并保持最新的路由信息。
* `维护路由表`：路由器每隔30秒发送更新报文，同时接收相邻路由器发送的更新报文以维护路由表项。
* `老化路由表项`：路由器为将自己构建的路由表项启动180秒的定时器。180秒内，如果路由器收到更新报文，则重置自己的更新定时器和老化定时器。
* `垃圾收集表项`：如果180秒过后，路由器没有收到相应路由表项的更新，则启动时长为120秒的垃圾收集定时器，同时将该路由表项的度量置为16。
* `删除路由表项`：如果60秒之后，路由器仍然没有收到相应路由表项的更新，则路由器将该表项删除

正常情况下，每30秒路由器就可以收到一次路由信息更新，如果经过180秒，即6个更新周期，一个路由项都没有得到更新，路由器就认为它已失效了。如果经过240秒，即8个更新周期，路由项仍没有得到更新，它就被从路由表中删除。

## 2 拓扑图

本篇文章涉及的拓扑如下：

![RIP学习拓扑](/images/network/rip/RIP拓扑图.png "本文涉及的实验拓扑")

* R1和R2通过单播建立邻居
* R1和R3使用组播建立邻居
* R2和R4通过广播建立邻居
* R4和R5通过不同版本建立邻居
* R4和R5启用密码认证的
* R4的传递的192.168.47.0/24路由Metric设置为6
* R3和R4不建立邻居
* R1发布默认路由，R3不能学到默认路由
* 当R1的lookback0中断后，不发布默认路由
* R4过滤从R2学到的10.1.13.0/24的路由
* R4汇总一条10.1.0.0/16的路由给R5

## 3 建立邻居（neighbors）

### 3.1 组播方式（Multicast）

R1和R3使用组播建立邻居，组播地址: `224.0.0.9`

```shell
# R1
interface GigabitEthernet2
 ip address 10.1.13.1 255.255.255.0

router rip
 version 2  # 设置RIP版本
 network 10.0.0.0 # 宣告IP地址属于此A类网络的接口参与RIP
 no auto-summary # 关闭自动汇总

# R3
interface GigabitEthernet2
 ip address 10.1.13.3 255.255.255.0

router rip
 version 2
 network 10.0.0.0
 no auto-summary
```

* `network`: 是用来宣告哪个网络会参与RIP，即该网络所在的Interface要发送RIP报文，报文内包含网络信息，不支持VLSM(可变长子网掩码)
* `version`: 变更RIP版本，默认版本是1, 这里改成2
* `auto-summary`: 关闭路由汇总，后边再介绍这个


### 3.2 单播方式（Unicast)

R1和R2使用单播建立邻居，网络不支持组播的时候可以选择这个

```shell
# R1
interface GigabitEthernet1
 ip address 10.1.12.1 255.255.255.0

router rip
 version 2
 passive-interface GigabitEthernet1
 network 10.0.0.0
 neighbor 10.1.12.2
 no auto-summary

# R2
interface GigabitEthernet1
 ip address 10.1.12.2 255.255.255.0

router rip
 version 2
 passive-interface GigabitEthernet1
 network 10.0.0.0
 neighbor 10.1.12.1
 no auto-summary
```
* `passive-interface`:  关闭`Ethernet0/0`口组播更新, 后边再说此命令
* `neighbor`: 设置单播地址

### 3.2 广播方式（Broadcast）

R2和R4使用广播创建邻居，很少用。

```shell
# R2
interface GigabitEthernet2
 ip address 10.1.24.2 255.255.255.0
 ip rip v2-broadcast

router rip
 version 2
 network 10.0.0.0
 no auto-summary

# R4
interface GigabitEthernet2
 ip address 10.1.13.3 255.255.255.0

router rip
 version 2
 network 10.0.0.0
 no auto-summary
```

* `ip rip v2-broadcast`: 指定此接口下使用广播方式来发送RIP报文

## 4 RIP的路由数据库（RIP Database）

```shell
# R3
R3#show ip rip database 
10.0.0.0/8    auto-summary
10.1.12.0/24
    [1] via 10.1.13.1, 00:00:06, GigabitEthernet2
10.1.13.0/24    directly connected, GigabitEthernet2
10.1.24.0/24
    [1] via 10.1.34.4, 00:00:04, GigabitEthernet3
10.1.34.0/24    directly connected, GigabitEthernet3
192.168.45.0/24    auto-summary
192.168.45.0/24
    [1] via 10.1.34.4, 00:00:04, GigabitEthernet3
```

每个运行 RIP的路由器管理一个路由数据库，该路由数据库包含了到所有可达目的地的路由项，这些路由项包含下列信息：

* 目的地址：主机或网络的 IP地址。
* 下一跳地址：为到达目的地，需要经过的本路由器相邻路由器的接口 IP地址。
* 出接口：本路由器转发报文的出接口。
* 度量值：本路由器到达目的地的开销。
* 路由时间：从路由项最后一次被更新到现在所经过的时间，路由项每次被更新时，路由时间重置为 0。

RIP第1版本不支持变长子网和非连续网络，RIP的第2版本和OSPF则支持变长子网和非连续网络。

## 5 防止路由环路（Prevent Routing Loops）

距离向量（Distance-Vector，D-V）类的算法容易产生路由循环，RIP是距离向量算法（Distance-Vector，D-V）的一种，所以它也不例外。如果网络上有路由循环，信息就会循环传递，永远不能到达目的地。为了避免这个问题，RIP等距离向量算法实现了下面4个机制。

* `水平分割（split horizon）`: 水平分割保证路由器记住每一条路由信息的来源，并且不在收到这条信息的端口上再次发送它。这是保证不产生路由循环的最基本措施。
* `毒性逆转（poison reverse）`: RIP从某个接口学到路由后，将该路由的度量值设置为16（不可达），并从原接口发回邻居路由器。利用这种方式，可以清除对方路由表中的无用信息。
* `触发更新（trigger update）`: 当路由表发生变化时，更新报文立即广播给相邻的所有路由器，而不是等待30秒的更新周期。同样，当一个路由器刚启动RIP时，它广播请求报文。收到此广播的相邻路由器立即应答一个更新报文，而不必等到下一个更新周期。这样，网络拓扑的变化会最快地在网络上传播开，减少了路由循环产生的可能性。
* `计数到无穷大（Count to Infinity）`: 即便采用了上面的几种方法，路由循环的问题也不能完全解决，只是得到了最大程度的减少。一旦路由循环真的出现，路由项的度量值就会出现计数到无穷大的情况。RIP选择16作为不可达的度量值是很巧妙的，它既足够的大，保证了多数网络能够正常运行，又足够小，使得计数到无穷大所花费的时间最短。

## 6 计时器（Timer）

RIP 一共有4个计时器（Timer）， 分别是更新计时器（Update Timer）、失效计时器（Invalid Timer）、抑制计时器（Hold-down Timer）和刷新计时器（Flush Timer)

* `Update Timer`: 多久发送一次更新报文，预设30秒
* `Invalid Timer`: 没有再收到某条路由的更新报文，Invalid Timer开始倒数，倒数至0后，就会把这条路由变更成失效（possibly down）状态，并将Metric设置成16， 然后将这条路有发送给其他邻居，这样做是希望其他路由器都知道这条路由出现问题，预设是180秒
* `Hold-down Timer`: 
  
  以下三种情况会将路由设置成Hold-down状态：
  
  1. Invalid timer 倒数到0
  2. 收到其他邻居告知这条路由的Metric是16（无法到达）
  3. 这条路由的度量值（Metric）变大了
  
  进入Hold-down状态的路由不会再接受任何更新，增加稳定性，预设是180秒

* `Flush Timer`: 当没有再收到某条路由的更新报文，刷新计时器开始倒数，倒数至0后，这条路由就会被删除，刷新计时器预设是240秒，注意Invalid Timer和Flush Timer是同时开始倒数的，Invalid Timer超时后再等60秒才删除路由

* 5.1 修改计时器

  **不建议修改，如果真的要改，请确保每一台路由器的timer都相同，否则会造成不稳定的问题**。可以用 `timers basic <update> <invalid> <hold-down> <flush>`命令修改RIP的计时器, 要按照`1:6:6:8`的比例来设置计时器时间

  ```shell
  conf t
  router rip
  timers basic 10 60 60 80
  ```

## 7 设置Metric

RIP的Metric用跳数（Hop Count）计算的，经过N台路由器，Metric就是N


由于RIP的Metric并非有Bandwidth计算出来的，因此不能通过更改Bandwidth来改变metric。可以使用offset来进行手动修改metric。

**示例**：

修改R4传递的的路由条目192.168.45.0/24的Metric值为6

```shell
# R4
conf t
access-list 1 permit 192.168.45.0 0.0.0.255

router rip
offset-list 1 out 5 # 宣告路由时把Metric增加5


# R5 
# 检查路由表中192.168.45.0/24的路由Metric变成6了
R5#show ip route rip
      10.0.0.0/8 is variably subnetted, 6 subnets, 2 masks
R        10.1.13.0/24 [120/1] via 10.1.12.1, 00:00:01, GigabitEthernet1
R        10.1.34.0/24 [120/1] via 10.1.24.4, 00:00:10, GigabitEthernet2
R     192.168.45.0/24 [120/6] via 10.1.24.4, 00:00:10, GigabitEthernet2
```

* `Hop Count`: 经过几台路由器到达目的地，最大为15，16或以上会被认为无法到达（Unreachable），

* `offset-list`: 增加宣告出去的路由条目的Metric值；RIP的Metric如果是16或以上会被视为Unreachable，所以只要将Metric设置为16，也可以做到路由过滤的效果

## 8 路由汇总

随着网络规模不断增大，路由条目亦会不断增加，路由汇总可以把路由组合，以减少路由条目，从而减低路由器的CPU、内存和带宽的消耗。在网络设置实施过程中切忌不能随意应用路由汇总，路由汇总只有再有合理设计及规划的基础上进行应用

路由表中没有属于汇总路由的子网路由条目，那么这个汇总路由就不会传递给邻居，这样保证了网络翻滚，优化性能。

### 7.1 自动汇总（Auto Summary）

* RIP默认开启自动汇总的
* 自动汇总（Auto Summary）会使用预设的Subnet Mask, Class A IP就用/8，Class B IP就用/16， Class C IP就用/24来进行路由汇总，自动汇总会路由中网路地址变大。
* 自动汇总是当发送的时候满足汇总条件，才进行自动汇总，所以自动汇总的路由条目不会回传。
* 由于自动汇总不仅汇总本地的路由，还会汇总邻居传递过来的路由，容易产生路由黑洞，所以不建议使用自动汇总，可以采用手工汇总的方式
* 关闭自动汇总的命令

  ```shell
  conf t
  router rip
  no auto-summary
  ```

### 7.2 手工汇总（Summarization）

* 自动汇总路由比手工汇总路由的优先级高，所以在进行手动汇总路由的时候，需要先关闭自动汇总
* 手工汇总2的N次方条路由为一条汇总路由条目
* 手工汇总路由的跳数（Metric）使用汇总前路由的最小跳数（Metric）
* 手工汇总路由的方法：

  R4汇总R1、R2和R3的传过来的路由条目，然后传递给R5；RIP的手动汇总路由的命令是再接口下的

  ```shell
  # R4
  interface GigabitEthernet1
    ip address 192.168.45.4 255.255.255.0
    # 手工汇总路由
    ip summary-address rip 10.1.0.0 255.255.0.0
  
  router rip
    version 2
    network 192.168.45.0  # 宣告属于此C类网络的接口参与RIP
    no auto-summary  # 关闭自动汇总

  # R5
  # 直接受到了汇总路由
  R5#show ip route rip              
  Gateway of last resort is not set

        10.0.0.0/16 is subnetted, 1 subnets
  R        10.1.0.0 [120/1] via 192.168.45.4, 00:00:25, GigabitEthernet1
  ```

## 9 默认路由（Default Route）

* 可以使用RIP发布默认路由，假如R1发布默认路由给其他路由器，配置如下：

  ```shell
  # R1
  conf t
  router rip
  default-information originate

  # R5
  R5#show ip route rip

  R*    0.0.0.0/0 [120/3] via 192.168.45.4, 00:00:05, GigabitEthernet1
        10.0.0.0/24 is subnetted, 4 subnets
  R        10.1.12.0 [120/2] via 192.168.45.4, 00:00:05, GigabitEthernet1
  R        10.1.13.0 [120/3] via 192.168.45.4, 00:00:05, GigabitEthernet1
  R        10.1.24.0 [120/1] via 192.168.45.4, 00:00:05, GigabitEthernet1
  R        10.1.34.0 [120/1] via 192.168.45.4, 00:00:05, GigabitEthernet1
  ```

* 只在R1的Gi1口发布默认路由，Gi2口不下发默认路由，R3不会学习到默认路由

  ```shell
  # R1
  conf t
  route-map TO_R2 permit 10 
  set interface GigabitEthernet1

  router rip
  default-information originate route-map TO_R2

  # R3
  # 默认路由没有了
  R3#show ip route rip

        10.0.0.0/8 is variably subnetted, 6 subnets, 2 masks
  R        10.1.12.0/24 [120/1] via 10.1.13.1, 00:00:14, GigabitEthernet2
  R        10.1.24.0/24 [120/2] via 10.1.13.1, 00:00:14, GigabitEthernet2
  R     192.168.45.0/24 [120/8] via 10.1.13.1, 00:00:14, GigabitEthernet2
  ```

* 当R1的Loopback0中断，R1就停止发布默认路由

  ```shell
  # R1
  conf t
  interface Loopback1
  ip address 1.1.1.1 255.255.255.255

  ip prefix-list CHECKING seq 5 permit 1.1.1.1/32

  route-map TO_R2 permit 10 
  match ip address prefix-list CHECKING
  set interface GigabitEthernet1

  router rip
  default-information originate route-map TO_R2

  # 关闭R1的Loopback1接口
  conf t
  interface Loopback1
  shutdown

  # R2
  # 默认路由立即消失了，应该是R1发布了一条毒化路由
  R2#show ip route rip

        10.0.0.0/8 is variably subnetted, 6 subnets, 2 masks
  R        10.1.13.0/24 [120/1] via 10.1.12.1, 00:00:12, GigabitEthernet1
  R        10.1.34.0/24 [120/1] via 10.1.24.4, 00:00:17, GigabitEthernet2
  R     192.168.45.0/24 [120/6] via 10.1.24.4, 00:00:17, GigabitEthernet2
  ```


## 10 被动接口（Passive Interface）

* 使用network方式建立邻居连接的时候，会通过所有匹配的接口发送路由更新组播报文，包括loopback口
* 可以将一个接口设置成被动接口，让它不发送路由更新组播报文
* 被动接口还是可以收到广播、组播和单播的路由更新报文
* 被动接口的网络也会添加到路由更新报文中，发送给RIP邻居
* 被动接口最好再两端都做，由于被动接口只接收不发送路由更新报文，会导致业务数据包来回路径不一致
* 配置示例，R3和R4不建立邻居，R3无法通过R4学到路由
  
  ```shell
  # R3
  router rip
    passive-interface default  # 设置所有接口为被动接口
    no passive-interface GigabitEthernet2  # 排除Gi2口

  # R4
  router rip
    passive-interface GigabitEthernet3  # 设置Gi3口为被动接口
  ```

## 11 路由过滤（Route Filtering）

限制RIP Route只发布某些路由，也可以控制接收方选择接收哪些路由，设定方式主要有`Distribute-list`、`Distance`和`Offset-list`三种，

使用者三种方式设置R4过滤从R2学到的10.1.13.0/24的路由的设置如下

* `Distribute-list`: 
  
  ```shell
  # R2
  conf t
  # 这里使用扩展列表，其实使用标准列表或者prefix-list更简单
  ip access-list extended 100
    deny   ip host 0.0.0.0 10.1.13.0 0.0.0.255  # 由于是再R2的OUT方向做的ACL，所以源地址是0.0.0.0
    permit ip any any

  router rip
    distribute-list 100 out GigabitEthernet2

  # R4 
  # 10.1.13.0/24的路由条目等待240秒后消失了
  R4#show ip route rip

  R*    0.0.0.0/0 [120/2] via 10.1.24.2, 00:00:14, GigabitEthernet2
        10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
  R        10.1.12.0/24 [120/1] via 10.1.24.2, 00:00:14, GigabitEthernet2
  ```

* `Distance`: 路由不会将AD为255的路由加入路由表，所以可以通过更改学到的路由AD来达到过滤的效果，不推荐

  ```shell
  # R4
  conf t
  ip access-list standard AD_255
  permit 10.1.13.0 0.0.0.255
  router rip
  distance 250 10.1.24.2 0.0.0.0 AD_255 # 我这里修改AD为250，方便看到修改后的变化，如果要过滤，修改成255就好了

  # R4的路由表
  R4#show ip route rip

  R*    0.0.0.0/0 [120/2] via 10.1.24.2, 00:00:13, GigabitEthernet2
        10.0.0.0/8 is variably subnetted, 6 subnets, 2 masks
  R        10.1.12.0/24 [120/1] via 10.1.24.2, 00:00:13, GigabitEthernet2
  R        10.1.13.0/24 [250/2] via 10.1.24.2, 00:00:13, GigabitEthernet2
  ```

* `Offset-list`: 在`设置Metric`的时候已经介绍过了，只需要将Metric修改为16或以上就被视为无效，这样可以做到过滤的效果，推荐用于限制传播距离的时候使用。

## 12 密码认证（Authentication）

RIP支持路由更新中包含密码，两端的密码都一样才可以进行路由交换,设置密码认证需要两个步骤，如下，这里设置的是R4和R5需要密码认证
* 设定钥匙串(Key Chain)
  
  ```shell
  # R4
  key chain CCIE # 定义一个钥匙串
    key 1  # 钥匙的ID
      key-string CCIE  # 要是的密码

  interface GigabitEthernet1
    ip rip authentication mode md5 # 密码的加密方式，只能MD5加密
    ip rip authentication key-chain CCIE # 配置这个接口需要密码认证，并调用这个钥匙串

  # R5
  key chain CCIE
    key 1
      key-string CCIE
  
  interface GigabitEthernet1
    ip rip authentication mode md5
    ip rip authentication key-chain CCIE
  ```

* 在Interface调用钥匙串，并设置加密方法

## 13 兼容性（compatible）

当两台设备中，有一台不支持RIP V2，他们俩启用RIP协议的话，需要考虑兼容性问题，可以在高版本的路由器上，配置接口能同时收发RIP V1和V2的报文，来解决兼容性问题，这里将以R4是V2,R5是V1来做演示

```shell
# R4 配置接口可以同时收发V1和V2版本的路由更新
conf t
interface GigabitEthernet1
 ip rip send version 1 2
 ip rip receive version 1 2

# R5
# 可以看到前边在接口下做的汇总路由失效了，是因为V1版本不支持手动汇总（VLSM的原因）
R5#show ip route rip
R*    0.0.0.0/0 [120/3] via 192.168.45.4, 00:00:19, GigabitEthernet1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
R        10.0.0.0/8 [120/1] via 192.168.45.4, 00:00:19, GigabitEthernet1
R        10.1.0.0/16 is possibly down,
          routing via 192.168.45.4, Gigab
```

## 14 RIP与BFD联动

双向转发检测BFD（Bidirectional Forwarding Detection）是一种用于检测邻居路由器之间链路故障的检测机制，它通常与路由协议联动，通过快速感知链路故障并通告使得路由协议能够快速地重新收敛，从而减少由于拓扑变化导致的流量丢失。在RIP与BFD联动中，BFD可以快速检测到链路故障并通知RIP协议，从而加快RIP协议对于网络拓扑变化的响应。

配置实例，R1和R2配置RIP与BFD联动

```shell
# R1
conf t
interface GigabitEthernet1
ip address 10.1.12.1 255.255.255.0
bfd interval 50 min_rx 50 multiplier 5  # 配置BFD

router rip
version 2
passive-interface GigabitEthernet1
network 10.0.0.0
neighbor 10.1.12.2 bfd # 配置RIP和BFD联动
no auto-summary

# R2
conf t
interface GigabitEthernet1
ip address 10.1.12.2 255.255.255.0
bfd interval 50 min_rx 50 multiplier 5

router rip
version 2
passive-interface GigabitEthernet1
network 10.0.0.0
neighbor 10.1.12.1 bfd
no auto-summary

# 检查
R2#show ip rip neighbors 
BFD Sessions created for RIP neighbors
Neighbor         Interface                      SessionHandle
10.1.12.1        GigabitEthernet1               1 

R2#show bfd neighbors
IPv4 Sessions
NeighAddr                              LD/RD         RH/RS     State     Int
10.1.12.1                            4097/4097       Up        Up        Gi1
```

## 15 路由重发布（Redistribute）

可以将其他动态路由协议、静态路由或者直连重发布到RIP中，实现多种路由协议之间协同工作

配置实例：将R1的loopback0重发布到RIP中

```shell
interface Loopback1
 ip address 1.1.1.1 255.255.255.255

router rip
 redistribute connected 
 # 重发布静态，也可以重分布OSPF、EIGRP等其他路由协议的路由
 # 此命令后边可以调用route-map，对重分布的路由条目进行过滤
```

## 16 故障处理（Troubleshooting）

* 检查接口是否在RIP中使能：使用show ip rip database  可以查看运行rip的接口；
* 检查对方发送版本号和本地接口接收的版本号是否匹配：缺省情况下，接口只发送RIPv1报文，但可以接收RIPv1和RIPv2报文。当入接口与收到的RIP报文使用不同的版本号时，有可能造成RIP路由不能被正确的接收；
* 检查在RIP中是否配置了策略，过滤掉收到的RIP路由：如果被路由策略过滤掉，则需修改路由策略；
* RIP使用的端口520/UDP是否被禁用；
* 检查接口是否配置了undo rip input/output或者rip metricin设置度量值过大；
* 检查接口是否配置了抑制接口；
* 检查路由度量值是否大于16；
* 检查链路两端是否配置了认证，认证的配置是否正确。


## 17 参考连接

* [H3C RIP技术介绍](http://www.h3c.com/cn/d_200805/605876_30003_0.htm)
* [曹世宏博客 RIP基础知识](https://cshihong.github.io/2018/03/23/RIP%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)
* [Jan Ho的网络世界](https://www.jannet.hk/routing-information-protocol-rip-zh-hans/)