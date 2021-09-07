---
title: "iptbales学习笔记"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['linux防火墙', 'linux firewall','centos防火墙']
description: "本文介绍了通过iptables来配置linux防火墙,最新的centos8 已经不适用此文档"
tags: ['linux']
categories: ['linux']
---

本文介绍了通过iptables来配置linux防火墙,最新的centos8 已经不适用此文档
<!--more-->

# iptbales配置

## iptables介绍

> 包过滤型的防火墙，隔离工具；工作于主机或网络边缘，对于进出本主机或本网络的报文根据事先定义的检查规则作匹配检测，对于能够被规则匹配到的报文作出相应处理的组件；
> `iptables` 的底层是用内核的`netfilter` 来进行包过滤的

* `netfilter hook function`:

  * `prerouting`
  * `input`
  * `output`
  * `forward`
  * `postrouting`

* 链（内置）：`PREROUTING`、`INPUT`、`FORWARD`、`INPUT`、`POSTROUTING`
* 功能：
  * `filter`：过滤，防火墙；
  * `nat`: `network address translation`；用于修改源IP或目标IP，也可以改端口；
  * `mangle`: 拆解报文，做出修改，并重新封装起来；
  * `raw`: 关闭nat表上启用的连接追踪机制；
* 功能<--链：
  * `raw`：`PREROUTING`， `OUTPUT`
  * `mangle`：`PREROUTING`，`INPUT`，`FORWARD`，`OUTPUT`，`POSTROUTING`
  * `nat`：`PREROUTING`，`[INPUT]OUTPUT`，`POSTROUTING`
  * `filter`：`INPUT`，`FORWARD`，`OUTPUT`
* 报文流向：
  * 流入本机：`PREROUTING` --> `INPUT`
  * 由本机流出：`OUTPUT` --> `POSTROUTING`
  * 转发：`PREROUTING` --> `FORWARD` --> `POSTROUTING`
* 路由功能发生的时刻：
  * 报文进入本机后：
    判断目标主机是？
  * 报文离开本机之前：
    判断经由哪个接口送往下一站？
* 规则
  * 组成部分：根据规则匹配条件来尝试匹配报文，一旦匹配成功，就由规则定义的处理动作作出处理；
    * 匹配条件：
      * 基本匹配条件：内建
      * 扩展匹配条件：由扩展模块定义；
    * 处理动作：
      * 基本处理动作：内建
      * 扩展处理动作：由扩展模块定义；
      * 自定义处理机制：自定义链
  * iptables的链：内置链和自定义链
    * 内置链：对应于hook function
    * 自定义链接：用于内置链的扩展和补充，可实现更灵活的规则管理机制；
  * 添加规则时的考量点：
    * 要实现哪种功能：判断添加到哪个表上；
    * 报文流经的路径：判断添加到哪个链上；
    * 链：链上的规则次序，即为检查的次序；因此，隐含一定的应用法则：
      * 同类规则（访问同一应用），匹配范围小的放上面；
      * 不同类的规则（访问不同应用），匹配到报文频率较大的放在上面；
      * 将那些可由一条规则描述的多个规则合并起来；
      * 设置默认策略；

### iptables命令

> 高度模块化，由诸多扩展模块实现其检查条件或处理动作的定义；模块位置`/usr/lib64/xtables/`，
> `IPv6`模块`libip6t_`开头，`IPv4`模块`libipt_`, `libxt_`开头

```sh
iptables [-t table] {-A|-C|-D} chain rule-specification
iptables [-t table] -I chain [rulenum] rule-specification
iptables [-t table] -R chain rulenum rule-specification
iptables [-t table] -D chain rulenum
iptables [-t table] -S [chain [rulenum]]
iptables [-t table] {-F|-L|-Z} [chain [rulenum]] [options...]
iptables [-t table] -N chain
iptables [-t table] -X  [chain]
iptables [-t table] -P chain target
iptables [-t table] -E old-chain-name new-chain-name
rule-specification = [matches...]  [target]
match = -m matchname [per-match-options]
target = -j targetname [per-target-options]
```

* 规则格式：

    ```sh
    iptables [-t table] COMMAND chain [PARAMETERS] [-m matchname [per-match-options]] -j targetname [per-target-options]
    ```

  * `-t table`:
        raw, mangle, nat, [filter]
  * `COMMAND`:
    * 链管理:

        ```sh
        -N：new, 自定义一条新的规则链；
        -X： delete，删除自定义的规则链；
            注意：仅能删除 用户自定义的 引用计数为0的 空的 链；
        -P：Policy，设置默认策略；对filter表中的链而言，其默认策略有：
            ACCEPT：接受
            DROP：丢弃
            REJECT：拒绝
        -E：重命名自定义链；引用计数不为0的自定义链不能够被重命名，也不能被删除；
        ```

    * 规则管理:

        ```sh
        -A：append，追加；
        -I：insert, 插入，要指明位置，省略时表示第一条；
        -D：delete，删除；
            (1) 指明规则序号；
            (2) 指明规则本身；
        -R：replace，替换指定链上的指定规则；
        -F：flush，清空指定的规则链；
        -Z：zero，置零；
            iptables的每条规则都有两个计数器：
                (1) 匹配到的报文的个数；
                (2) 匹配到的所有报文的大小之和；
        ```

    * 查看

        ```sh
        -L：list, 列出指定鏈上的所有规则；
        -n：numberic，以数字格式显示地址和端口号；
        -v：verbose，详细信息；
            -vv, -vvv
        -x：exactly，显示计数器结果的精确值；
        --line-numbers：显示规则的序号；
        ```

  * `chain`：
    * 内置链
        PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING
    * 自定义链
  * 匹配条件：
    * 基本匹配条件(```PARAMETERS```)：
        > 无需加载任何模块，由iptables/netfilter自行提供；
        > `[!]`取反

        ```sh
        [!] -s, --source  address[/mask][,...]：检查报文中的源IP地址是否符合此处指定的地址或范围；
        [!] -d, --destination address[/mask][,...]：检查报文中的目标IP地址是否符合此处指定的地址或范围；所有地址：0.0.0.0/0
        [!] -p, --protocol protocol: tcp, udp, udplite, icmp, icmpv6,esp, ah, sctp, mh or  [all]
        [!] -i, --in-interface name：数据报文流入的接口；只能应用于数据报文流入的环节，只能应用于PREROUTING，INPUT和FORWARD链；
        [!] -o, --out-interface name：数据报文流出的接口；只能应用于数据报文流出的环节，只能应用于FORWARD、OUTPUT和POSTROUTING链；
        ```

    * 扩展匹配(```matchname```)
      * 隐式扩展
        > 在使用-p选项指明了特定的协议时，无需再同时使用-m选项指明扩展模块的扩展机制；
        * `tcp`:

            ```sh
            [!] --source-port, --sport port[:port]：匹配报文的源端口；可以是端口范围；
            [!] --destination-port,--dport port[:port]：匹配报文的目标端口；可以是端口范围；
            [!] --tcp-flags  mask  comp
                    mask is the flags which we should examine,  written as a comma-separated list，例如 SYN,ACK,FIN,RST
                    comp is a comma-separated list  of  flags  which must be set，例如SYN
                    例如：“--tcp-flags  SYN,ACK,FIN,RST  SYN”表示，要检查的标志位为SYN,ACK,FIN,RST四个，其中SYN必须为1，余下的必须为0；
            [!] --syn：用于匹配第一次握手，相当于”--tcp-flags  SYN,ACK,FIN,RST  SYN“；
            ```

        * `udp`:

            ```sh
            [!] --source-port, --sport port[:port]：匹配报文的源端口；可以是端口范围；
            [!] --destination-port,--dport port[:port]：匹配报文的目标端口；可以是端口范围；
            ```

        * `icmp`:

            ```sh
            [!] --icmp-type {type[/code]|typename}
                echo-request：8/0
                echo-reply：0/0
            ```

      * 显式扩展
        > 必须使用-m选项指明要调用的扩展模块的扩展机制；
        * `multiport`:
            > 以离散或连续的 方式定义多端口匹配条件，最多15个；

            ```sh
            [!] --source-ports,--sports port[,port|,port:port]...：指定多个源端口；
            [!] --destination-ports,--dports port[,port|,port:port]...：指定多个目标端口；
            $ sudo iptables -I INPUT  -d 172.16.0.7 -p tcp -m multiport --dports 22,80,139,445,3306 -j ACCEPT
            ```

        * `iprange`:
            > 以连续地址块的方式来指明多IP地址匹配条件；

            ```sh
            [!] --src-range from[-to]
            [!] --dst-range from[-to]
            $ sudo iptables -I INPUT -d 172.16.0.7 -p tcp -m multiport --dports 22,80,139,445,3306 -m iprange --src-range 172.16.0.61-172.16.0.70 -j REJECT
            ```

        * `time`:
            > 如果数据包到达时间/日期在给定范围内，则匹配。

            ```sh
            --timestart hh:mm[:ss]
            --timestop hh:mm[:ss]
            [!] --weekdays day[,day...]
            [!] --monthdays day[,day...]
            --datestart YYYY[-MM[-DD[Thh[:mm[:ss]]]]]
            --datestop YYYY[-MM[-DD[Thh[:mm[:ss]]]]]
            --kerneltz：使用内核配置的时区而非默认的UTC；
            ```

        * `string`:
            > 此模块使用某种模式匹配策略来匹配给定的字符串, 只对明文编码的字符串可以匹配

            ```sh
            --algo {bm|kmp}             # 检测模式
            [!] --string pattern        # 字符串
            [!] --hex-string pattern    # 字符串hex
            --from offset               # 从数据包的那里开始检测
            --to offset                 # 到数据包的那里检测结束
            $ sudo iptables -I OUTPUT -m string --algo bm --string "gay" -j REJECT
            ```

        * `connlimit`:
            > 允许您限制每个客户端IP地址（或客户端地址块）到服务器的并行连接数。

            ```sh
            --connlimit-upto n      # 上限，小于等于N
            --connlimit-above n     # 下限，大于N
            $ sudo iptables -I INPUT -d 172.16.0.7 -p tcp --syn --dport 22 -m connlimit --connlimit-above 2 -j REJECT
            ```

        * `limit`:
          > 该模块使用令牌桶过滤器以有限的速率匹配。
          > 限制本机某tcp服务接收新请求的速率：--syn, -m limit

          ```sh
          # 限制每(秒|分钟|小时|天）可以生成多少个令牌
          --limit rate[/second|/minute|/hour|/day]
          # 令牌桶最大可以存放多少个令牌（一个令牌代表一个数据包）
          --limit-burst number
          ```

        * `state`:
        * 

  * 处理动作：
    * ```-j targetname [per-target-options]```
      * 简单`target`：
        `ACCEPT`, `DROP`, `REJECT`
      * 扩展`target`：
