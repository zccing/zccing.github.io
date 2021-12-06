---
title: "linux系统基准测试工具sysbenchtest的安装和使用"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['sysbenchtest', '基准测试工具','linux基准测试']
description: "linux系统基准测试工具sysbenchtest的安装和使用"
tags: ['linux']
categories: ['linux']
---

linux系统基准测试工具sysbenchtest的安装和使用
<!--more-->


# Sysbench 基准测试工具

## Sysbench安装

### 二进制安装

* centos
    ```shell
    yum install epel-release    # 安装centos社区源
    yum install sysbench        # 安装sysbench
    ```
* ubuntu
    ```shell
    curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash  #
    sudo apt -y install sysbench
    ```

### 编译安装

> github编译安装介绍：<https://github.com/akopytov/sysbench#building-and-installing-from-source>

* 下载源代码
    github地址：<https://github.com/akopytov/sysbench/releases>
    ```shell
    wget https://github.com/akopytov/sysbench/archive/1.0.16.tar.gz
    tar -zxvf 1.0.16.tar.gz
    ```

* 安装依赖
  * centos
    ```shell
    yum -y install make automake libtool pkgconfig libaio-devel
    # For MySQL support, replace with mysql-devel on RHEL/CentOS 5
    yum -y install mariadb-devel openssl-devel
    # For PostgreSQL support
    yum -y install postgresql-devel
    ```
  * ubuntu
    ```shell
    apt -y install make automake libtool pkg-config libaio-dev
    # For MySQL support
    apt -y install libmysqlclient-dev libssl-dev
    # For PostgreSQL support
    apt -y install libpq-dev
    ```
* 编译安装
    ```shell
    ./autogen.sh
    # Add --with-pgsql to build with PostgreSQL support
    ./configure
    make -j
    make install
    ```

## 参数解析

* 一般语法
    sysbench的常规命令行语法是：

    ```shell
    sysbench [options] ... [testname] [command]
    ```

  * `testname` 是内置测试的可选名称(例如`fileio`，`memory`，`cpu`,`threads`,`mutex`等)，或捆绑的Lua脚本之一的名称（例如`oltp_read_only`, 或自定义Lua脚本的`path`。
    如果不测试名称在命令行中指定（因此，没有`command`也是如此，因为它将被解析为`testname`），或测试名称是破折号(" - ")，然后sysbench期望Lua脚本执行其标准输入。

  * `command` 是一个可选参数，将由sysbench传递给用`testname`指定的内置测试或脚本。
    `command`定义必须由测试执行的`action`。清单可用命令取决于特定测试。一些测试也实现自己的自定义命令。

    以下是典型测试命令及其用途的说明：

    * `prepare`：为需要的测试执行准备操作它们，例如在磁盘上为`fileio`创建必要的文件测试或填充测试数据库以获取数据库基准。
    * `run`：运行用`testname`指定的实际测试论点。所有测试都提供此命令。
    * `cleanup`：在测试运行后删除临时数据创建一个的测试。
    * `help`：显示使用指定的测试的使用信息
    * `testname`: 参数。这包括完整的命令列表由测试提供，因此它应该用于获得可用命令。

  * `options` 是0个或多个命令行选项的列表。与`command`一样，`sysbench testname help`命令应该用来描述a提供的可用选项特别测试。
    请参见`常规命令行选项`有关sysbench本身提供的常规选项的说明。
    您可以使用`sysbench --help`来显示常规命令行语法和选项。

* 常规命令行选项

    | 选项                    | 说明                                                                                                                             | *默认值* |
    | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------- |
    | `--threads`             | 要创建的工作线程总数                                                                                                             | 1        |
    | `--events`              | 请求总数限制(一个循环即一个`event`,这个定义执行多少个`event`)。0（默认值）表示没有限制                                           | 0        |
    | `--time`                | 限制总执行时间（以秒为单位）。0表示没有限制                                                                                      | 10       |
    | `--warmup-time`         | 执行多少秒的`events`后再启用实际基准测试统计信息，n秒之前禁用实际基准测试统计信息,                                               | 0        |
    | `--rate`                | 该数字指定平均每个线程应执行的每秒`event`数。0（默认值）表示无限速率，即事件尽可能快地执行                                       | 0        |
    | `--thread-init-timeout` | 工作线程初始化的等待时间（以秒为单位）                                                                                           | 30       |
    | `--thread-stack-size`   | 每个线程的堆栈大小                                                                                                               | 32K      |
    | `--report-interval`     | 定期报告具有指定间隔（以秒为单位）的中间统计信息 请注意，此选项生成的统计信息是按时间间隔而不是累积。0禁用中间报告               | 0        |
    | `--debug`               | 打印更多调试信息                                                                                                                 | 关       |
    | `--validate`            | 尽可能执行测试结果的验证                                                                                                         | 关       |
    | `--help`                | 打印有关一般语法或指定测试的帮助，然后退出                                                                                       | 关       |
    | `--verbosity`           | 详细级别（0  - 仅关键消息，5  - 调试）                                                                                           | 4        |
    | `--percentile`          | 此选项允许指定要计数的查询执行时间的百分位数等级， （例如，95％百分位意味着我们应该减少5％的最长请求并从剩余的请求中选择最大值） | 95       |
    | `--luajit-cmd`          | 执行LuaJIT控制命令。这个选项相当于`luajit -j`。有关更多信息，请参阅[LuaJIT文档]<http://luajit.org/running.html#opt_j>            |          |

    请注意，可以通过附加相应的乘法后缀（K表示千字节，M表示兆字节，G表示千兆字节，T表示太字节）来指定所有大小选项（--thread-stack-size如此表中）的数值。

* 随机数选项

    | 选项                 | 说明                                                                                                                              | *默认值* |
    | -------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------- |
    | `--rand-type`        | 随机数字分布{uniform，gaussian，special，pareto，zipfian}默认使用。基准脚本可以选择使用默认分发，或者明确指定它，即覆盖默认分配。 | 特别     |
    | `--rand-seed`        | 随机数发生器的种子。当为0时，当前时间用作RNG种子。                                                                                | 0        |
    | `--rand-spec-iter`   | 特殊分布的迭代次数                                                                                                                | 12       |
    | `--rand-spec-pct`    | 特殊分布中“特殊”值落入的整个范围的百分比 1                                                                                      |
    | `--rand-spec-res`    | 用于特殊分发的'特殊'值的百分比                                                                                                    | 75       |
    | `--rand-pareto-h`    | 帕累托分布的形状参数                                                                                                              | 0.2      |
    | `--rand-zipfian-exp` | Zipfian分布的形状参数（theta）                                                                                                    | 0.8      |

## CPU测试

> sysbench的cpu测试是在指定时间内，循环进行素数计算
> 素数（也叫质数）就是从1开始的自然数中，无法被整除的数，比如2、3、5、7、11、13、17等。编程公式：对正整数n，如果用2到根号n之间的所有整数去除，均无法整除，则n为素数。
> 循环计算`cpu-max-prime`定义的素数为一个`event`

```shell
# 素数上限2万，默认10秒，1个线程
sysbench cpu --cpu-max-prime=20000  --threads=1 run
```

* `--cpu-max-prime`
    素数生成数量的上限,

## 内存测试

> 测试内存的联系读写性能
> 读或者写一个`memory-block-size`大小，即一个`event`

```shell
# 使用8k的内存块测试1G的数据
sysbench memory --memory-block-size=8k --memory-total-size=10G run
```

* `memory-block-size`
    测试时内存块大小。默认是1K
* `memory-total-size`
    传输数据的总大小。默认是100G
* `memory-scope`
    内存访问范围{global,local}。默认是global
* `memory-hugetlb[=on|off]`
    从HugeTLB池内存分配。默认是off
* `memory-oper`
    内存操作类型。{read, write, none} 默认是write
* `memory-access-mode`
    存储器存取方式{seq,rnd} 默认是seq

## 文件I/O测试

> 磁盘IO性能测试[主要看每秒请求数(request)和总体的吞吐量(total)
> 一次IO请求为一个`event`

```shell
# 准备测试文件
sysbench fileio --threads=2 --file-total-size=2G prepare
# 测试
sysbench fileio --threads=2 --file-total-size=2G --file-test-mode=rndrw --time=10 run
# 删除测试数据
sysbench fileio --threads=2 --file-total-size=2G cleanup
```

* file-num=N                  创建测试文件的数量。默认是128
* file-block-size=N           测试时文件块的大小。默认是16384(16K)
* file-total-size=SIZE        测试文件的总大小。默认是2G
* file-test-mode=STRING       文件测试模式{seqwr(顺序写), seqrewr(顺序读写), seqrd(顺序读), rndrd(随机读), rndwr(随机写), rndrw(随机读写)}
* file-io-mode=STRING         文件操作模式{sync(同步),async(异步),fastmmap(快速map映射),slowmmap(慢map映射)}。默认是sync
* file-async-backlog=N        每个线程排队的异步操作数默认是128
* file-extra-flags=[LIST,...] 使用额外的标志来打开文件{sync(同步),async(异步),direct} 。默认为空
* file-fsync-freq=N           在这个请求数之后执行fsync()。(0 – 不使用fsync())。默认是100
* file-fsync-all[=on|off]     每执行完一次写操作就执行一次fsync。默认是off
* file-fsync-end[=on|off]     在测试结束时才执行fsync。默认是on
* file-fsync-mode=STRING      使用哪种方法进行同步{fsync, fdatasync}。默认是fsync
* file-merged-requests=N      如果可以，合并最多的I/O请求数(0 – 表示不合并)。默认是0
* file-rw-ratio=N             测试时的读写比例。默认是1.5

## 线程基准测试

> thread-locks小于线程数除以2，lock越少，处理时间越长 

```shell
sysbench threads --threads=2 --thread-yields=2000 --thread-locks=10 run
```

* thread-yields=N 每个请求产生多少个线程。默认是1000
* thread-locks=N  每个线程的锁的数量。默认是8

## 互斥锁基准测试

> 使用互斥锁工作负载时，sysbench应用程序将为每个线程运行单个请求。此请求将首先对CPU施加压力（使用简单的增量循环，通过`--mutex-loops`参数），之后它将采用随机互斥（锁定），递增全局变量并再次释放锁定。通过lock（`--mutex-locks`）的数量重复该过程若干次。随机互斥锁取自`--mutex-num`参数大小的池。
> 此处运行的持续时间很重要，但必须考虑到线程将从可用池中获取随机互斥锁。这个随机因素可能会对结果产生一些影响。

```shell
sysbench mutex --threads=100 --mutex-num=1000 --mutex-locks=100000 --mutex-loops=10000 run
```

* mutex-num=N      数组互斥的总大小。默认是4096
* mutex-locks=N    每个线程互斥锁的数量。默认是50000
* mutex-loops=N    内部互斥锁的空循环数量。默认是10000

## sql测试（等用到的时候在做笔记）

## 结果解析

```echo
General statistics:
    total time:                          10.0024s
    total number of events:              3467

Latency (ms):
         min:                                    2.84
         avg:                                    2.88
         max:                                    8.98
         95th percentile:                        2.91
         sum:                                10000.21

Threads fairness:
    events (avg/stddev):           3467.0000/0.00
    execution time (avg/stddev):   10.0002/0.00
```

* `Latency`
    > 一般统计,单位:ms
  * `total time`
    执行测试花费的总时间
  * `total number of events`
    测试总共执行了多少个`event`

* `Latency (ms)`
    > 单个请求统计
  * `min`
    执行单个`event`的最小时间，单位：ms
  * `avg`
    执行所有`event`的平均时间，单位：ms
  * `max`
    执行单个`event`的最大时间，单位：ms
  * `95th percentile`
    95%次`event`完成的时间
  * `sum`
    执行的总共时间

* `Threads fairness`
  * `events (avg/stddev)`
    平均每个线程完成3255次event。`stddev`为线程之间的标准差，对于单线程无意义
  * `execution time (avg/stddev)`
    每个线程平均耗时多少秒。`stddev`为线程之间的标准差，对于单线程无意义

* 对比规则：
  * 相同时间，比较的是谁完成的`event`多
  * 相同`event`，比较的是谁用时更少
  * 时间和`event`都相同，比较`stddev`(标准差)
