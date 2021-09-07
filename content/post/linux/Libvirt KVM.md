---
title: "KVM libvirt学习笔记"
date: 2020-08-30T01:37:56+08:00
lastmod: 2020-09-5T01:37:56+08:00
draft: false
keywords: ['KVM', 'libvirt','虚拟机','linux虚拟机']
description: "介绍使用libvirt来管理Linux的KVM虚拟机，并介绍virsh命令"
tags: ['linux']
categories: ['linux']
---

使用libvirt来管理Linux的KVM虚拟机，主要使用virsh命令
<!--more-->

# kvm学习笔记

## 查看是否支持kvm虚拟化

1. CPU必需支持虚拟化，可以在/proc/cpuinfo文件中想找flags，如果是inter的显示为vmx，amd的显示为svm `cat /proc/cpuinfo | egrep "(vmx|svm)"`
2. CPU必需支持64位操作系统，可以在上述文件中查找lm标记，如果有则支持 `cat /proc/cpuinfo | egrep lm`
3. 系统必需为64为的RHEL，且系统版本为RHEL6.4及以上为最佳`uname -a`
4. 必需在BIOS里开启CPU的VT功能 `lsmod | grep kvm`

## 编译安装qemu和libvirt（未完成）

* 下载文件

    ```sh
    # 可以去官网下载最新的
    wget https://download.qemu.org/qemu-3.0.0.tar.xz
    ````

* 安装需要用到的库文件

    ```sh
    yum install git glib2-devel libfdt-devel pixman-devel zlib-devel bzip2-devel libaio-devel spice-server-devel spice-protocol libusb-devel usbredir-devel
    ```

* 编译安装

    ```sh
    ./configure \
    --prefix=/opt/qemu-kvm \
    --datadir=/home/data/kvm \
    --target-list=i386-softmmu,x86_64-softmmu \
    --enable-system \
    --disable-debug-info \
    --enable-usb-redir \
    --enable-libusb \
    --enable-spice \
    --enable-uuid \
    --enable-kvm \
    --enable-bzip2 \
    --enable-linux-aio \
    --enable-tools

    make -j4
    make install
    ```

* 创建环境变量和添加sytemd服务

## YUM安装

> centos默认存储库的版本过低，添加virt存储库，然后通过yum安装

* 添加qemu扩展存储库

    ```sh
    yum install centos-release-qemu-ev
    ```

* 安装qemu和libvirt

    ```sh
    yum install qemu-kvm libvirt -y
    ```

## libvirt 配置文件

> libvirt 的默认配置文件在：`/etc/libvirt`
> libvirt 的默认工作区在：`/var/cache/libvirt`

### libvirt 远程配置

### Guest xml配置文件介绍

> 验证xml文件`virt-xml-validate /path/to/XML/file`
> [官方文档](https://libvirt.org/drvqemu.html#xmlconfig)
> 主要介绍xml中每一行的意思，当然也可以手动创建适合您的`domain xml`文件然后通过`virsh define` 命令来定义一个虚拟机,然后通过`virsh start`来启动虚拟机

1. 通用根配置文件

    ```xml
    <domain type='kvm' id='1'>
        <name>MyGuest</name>
        <uuid>4dea22b3-1d52-d8f3-2516-782e98ab3fa0</uuid>
        <genid>43dc0cf8-809b-4adb-9bea-a9abb5f3d90e</genid>
        <title>A short description - title - of the domain</title>
        <description>Some human readable description</description>
        <metadata>
            <app1:foo xmlns:app1="http://app1.org/app1/">..</app1:foo>
            <app2:bar xmlns:app2="http://app1.org/app2/">..</app2:bar>
        </metadata>
        ......
    <domain>
    ```

    * 根元素`domain`有两个属性，`type`代表虚拟机的管理程序，有`kvm`，`xen`,`qemu`,`lxc`和`kqemu`；`id`代表正在运行的虚拟机唯一整数标识符，非活动的虚拟机没有id
    * `name`
        虚拟机的名称，同一个物理机器唯一
    * `uuid`
        全局唯一标识符，格式必须符合RFC 4122，如果在定义/创建新`domain`时省略，则会生成随机UUID。也可以通过[sysinfo](https://libvirt.org/formatdomain.html#elementsSysinfo)规范提供uuid
    * `genid`
        和`uuid`元素一样。
    * `title`
        可选元素title为虚拟机的简短描述，不应包含任何换行符。
    * `description`
        description元素为虚拟机的详细描述，libvirt不以任何方式使用此数据，它可以包含用户想要的任何信息
    * `metadata`（没理解）
        The metadata node can be used by applications to store custom metadata in the form of XML nodes/trees. Applications must use custom namespaces on their XML nodes/trees, with only one top-level element per namespace (if the application needs structure, they should have sub-elements to their namespace element)

2. 虚拟机系统启动相关

    > 有许多不同的方法可以启动虚拟机，每种方法各有利弊。

    1. bios启动
        > 通过BIOS引导可用于支持完全虚拟化的虚拟机管理程序。在这种情况下，BIOS具有引导顺序优先级（软盘，硬盘，cdrom，网络），用于确定获取/查找引导映像的位置。

        ```xml
        ...........
        <os>
            <type arch='x86_64' machine='pc-q35-rhel7.5.0'>hvm</type>
            <loader readonly='no' secure='no' type='rom'>/usr/share/OVMF/OVMF_CODE.fd</loader>
            <nvram>/home/cc/.config/libvirt/qemu/nvram/windows_VARS.fd</nvram>
            <boot dev='hd'/>
            <boot dev='cdrom'/>
            <bootmenu enable='yes' timeout='3000'/>
            <smbios mode='sysinfo'/>
            <bios useserial='yes' rebootTimeout='0'/>
        </os>
        ...........
        ```

        * `type`
            `type`元素的内容指定要在虚拟机中引导的操作系统的类型, `hvm`为借助qemu-kvm完全虚拟化，`arch`属性指定虚拟化的CPU架构，`machine`属性指定虚拟化的机器类型；可以通过 [`virsh capabilities`](https://libvirt.org/formatcaps.html) 查看`type`元素支持的内容
        * `loader`
            `loader`可选元素的内容是指定虚拟机守护进程的绝对路径，用户辅助虚拟机的创建。有两个可选属性，`readonly`: 值有yes|no，标记镜像是可写或者只读；`type`: 值有rom|pflash，以什么方式引导镜像，pflash可以加载UEFI镜像，`secure`控制是否开启安全启动
        * `nvram`
            `nvram`可选元素的表示虚拟uefi固件的文件位置，在`qemu.conf`文件中定义了，`template`属性会覆盖掉`qemu.conf`中的关于nvram的相应配置
        * `boot`
            `boot`元素的值：`fd`,`hd`,`cdrom`或`network`中的一个，用于指定引导设备，可以重复多次来设置设备引导的优先级列表，在磁盘等设备的部分也可以控制引导顺序，并且优先级比这里的高，也是官方推荐的。
        * `bootmenu`
            `bootmenu`元素定义是否在guest启动时启用交互式启动菜单，如果未指定，将使用管理程序的默认值
        * `smbios`
            `smbios`元素定义如何填充guest虚拟机中可见的SMBIOS信息，必选属性`mode`的值:`emulate`(让虚拟机管理程序生成所有值)，`host`(从主机的SMBIOS值复制块0和块1中的所有块，除了UUID)或者`sysinfo`(使用[sysinfo](https://libvirt.org/formatdomain.html#elementsSysinfo)元素中的值)
        * `bios`
            `bios`元素定义bios启动的设置，uefi启动无效。`useserial`属性：启用或禁用串行图形适配器，允许用户在串行端口上查看BIOS消息，需要定义[串口](https://libvirt.org/formatdomain.html#elementCharSerial); `rebootTimeout`属性：引导失败后多久重新启动，毫秒为单位，最大65535，`-1`禁止启动

    2. 主机启动加载器
        使用其他的启动加载器来启动IOS文件，安装系统或者直接启动系统，例如 `pygrubXen`：

        ```xml
        <bootloader>/usr/bin/pygrub</bootloader>
        <bootloader_args>--append single</bootloader_args>
        ```

        * bootloader:
            bootloader元素的内容提供了主机OS​​中引导加载程序可执行文件的完全路径。将运行此引导加载程序以选择要引导的内核
        * bootloader_args:
            可选bootloader_args元素参数传递给引导加载程序

    3. 直接内核启动
        安装新的客户操作系统时，直接从存储在主机操作系统中的内核和initrd启动通常很有用，允许将命令行参数直接传递给安装程序。此功能通常适用于para和full虚拟客户端
        参考官网:<https://libvirt.org/formatdomain.html#elementsOSKernel>

    4. linux容器启动
        使用基于容器的虚拟化而不是内核/启动映像启动域
        参考官网:<https://libvirt.org/formatdomain.html#elementsOSContainer>

    5. SMBIOS系统信息
        一些管理程序允许控制向客户呈现的系统信息（例如，SMBIOS字段可以由管理程序填充并通过客户机中的dmidecode命令进行检查）。可选sysinfo元素涵盖所有此类信息。
        参照官网:<https://libvirt.org/formatdomain.html#elementsSysinfo>

3. CPU分配

    ```xml
    <domain>
    ......
        <vcpu placement='static' cpuset="1-4,^3,6" current="1">2</vcpu>
        <vcpus>
            <vcpu id='0' enabled='yes' hotpluggable='no' order='1'/>
            <vcpu id='1' enabled='no' hotpluggable='yes'/>
        </vcpus>
    ......
    </domain>
    ```

    * `vcpu`
        此元素的内容定义为guest虚拟机操作系统分配的最大虚拟CPU数，必须介于1和虚拟机管理程序支持的最大值之间。最好不要超过实际CPU数量
      * `cpuset`
        可选属性cpuset是以逗号分隔的物理CPU编号列表，默认情况下可以固定域进程和虚拟CPU。该列表中的每个元素可以是单个CPU编号，一系列CPU编号，`^`后跟要从先前范围中排除的CPU编号。
      * `current`
        是否应启用最少CPU数量
      * `placement`
        placement(分配模式):static|auto，如果通过`cpuset`来指定CPU个数，必须为`static`。如果`placement`为`static`，`cpuset`并没有指定，将固定所有可用物理CPU
    * `vcpus`
        控制各个vCPU的状态
        * `ID`属性指定libvirt使用的vcpu id,注意：某些情况下，guest中现实的`vcpu id`可能和`libvirt`不一样，有效范围从`0`到由`vcpu`定义的最大CPU减一
        * `enable`属性表示控制`vcpu`的状态，值：`yes|no`；当值为`no`的时候开机不启用这个CPU，必须启动热插拔。
        * `hotpluggable`属性控制在启动时启用CPU的情况下，是否可以对指定的vCPU进行热插拔。请注意，所有已禁用的vCPU必须是可热插拔的(即`enable`是`no`的`hotpluggable`必须为`yes`)。有效值为 `yes`和`no`
        * `order`属性允许指定添加在线vCPU的顺序

4. IOThreads分配
    > IOThreads是专用的事件循环线程，用于支持的磁盘设备执行块I/O请求，以提高可扩展性，尤其是在具有许多LUN的SMP主机/ guest虚拟机上；有时间可以研究下。
    > [官方文档](https://libvirt.org/formatdomain.html#elementsIOThreadsAllocation)

5. CPU调整
    > 调整CPU的高级参数，配置的很少，可以参考官方文档，有时间可以研究下
    > [官方文档](https://libvirt.org/formatdomain.html#elementsCPUTuning)

6. 内存分配
    > 个人的一些理解

    ```xml
    <domain>
        ...
        <memory unit='KiB'>524288</memory>
        <maxMemory slots='16' unit='KiB'>1524288</maxMemory>
        <currentMemory unit='KiB'>524288</currentMemory>
        ...
    </domain>
    ```

    * memory
        `memory`元素定义了，guest虚拟机启动的时候最大内存，包括启动指定的内存和稍后添加的内存(后期通过`virsh`添加内存将自动把这里的的数值调大)，`unit`属性指定内存容量的单位，默认`KiB`,
        可选单位有：
        * 字节:`b` or `bytes`
        * 千字节:`KB` (1,000 bytes), `KiB` or `k` (1024 bytes)
        * 兆字节:`MB` (1,000,000 bytes), `M` or `Mib` (1,048,576 bytes)
        * 千兆字节:`GB` (1,073,741,824 bytes)
        * 太字节:`TB` (1,000,000,000,000 bytes), `T` or `TiB` (1,099,511,627,776 bytes)
        如果guest虚拟机配置了[NUMA](https://libvirt.org/formatdomain.html#elementsCPU),则`memory`元素可以省略，可选属性`dumpCore`用来控制内存崩溃的时候是否生成错误信息到coredump，值：`on` or `off`
    * maxMemory
        `maxMemory`元素定义guest通过`hot-plug`可以达到的最大内存，`unit`属性指定内存容量的单位，默认`KiB`, 可选单位和`memory`元素一样
        `slots`属性指定可用于向guest虚拟机添加内存的插槽数，每个插槽在运行时都可以插入一个内存设备，上限是 255 个。
        请注意，由于通过内存`hotplug`添加的内存块的对齐，可能无法实现此元素指定的完整大小分配。如果不需要内存热插拔，无需配置此选项
    * currentMemory
        `currentMemory`元素定义guest的实际分配的内存大小，`unit`属性与`memory`相同。如果省略将于`memory`一样的数值
        描述:
        在虚拟机启动后加载了内存balloon驱动后就开始对虚拟机内存进行热插拔，先设置内存为`currentMemory>`大小，这个`currentMemory`qemu进程不知道，是记录在libvirt中的。然后根据需求对内存进行调整（balloon），调整的上限是`memory`，这个`memory`qemu进程是知道的，在虚拟机启动时使用了这个值

7. 内存支持

    > 可选`memoryBacking`元素可能包含几个影响由主机页面支持虚拟内存页面的元素。

    ```xml
    <domain>
        ...
        <memoryBacking>
            <hugepages>
            <page size="1" unit="G" nodeset="0-3,5"/>
            <page size="2" unit="M" nodeset="4"/>
            </hugepages>
            <nosharepages/>
            <locked/>
            <source type="file|anonymous|memfd"/>
            <access mode="shared|private"/>
            <allocation mode="immediate|ondemand"/>
            <discard/>
        </memoryBacking>
        ...
    </domain>
    ```

    * `hugepages`
        > 定义guest虚拟机使用`hugepages`(大内存页面)，而不是使用本机使用的内存页面大小；
        > 在`numa`节点(即宿主机是numa节点)中可以使用`page`可选元素来具体的设置`numa`节点的大页面；`size`元素是必须的，定义使用的大页面的大小，默认单位为`Kib`；可选元素`ubit`定义`size`使用的单位；在`numa`的系统中，可选元素`nodeset`可以`NUMA`节点与某些大页面进行绑定，正确的语法，清参考`NUMA节点调整`
    * `nosharepages`
        告诉管理程序，禁止此`domain`不允许使用内存共享即`ksm`内存合并
    * `locked`
        当虚拟机管理程序设置和支持时，属于该`domain`的内存页面将被锁定在`host`的内存中，并且不允许`host`将它们交换出来，这可能是某些工作负载（如实时）所必需的。对于`QEMU/KVM guest`虚拟机，QEMU进程本身使用的内存也将被锁定：与`guest`虚拟机内存不同，这是libvirt无法预先确定的数量，因此它必须完全取消锁定内存的限制。因此，启用此选项会产生潜在的安全风险：`host`在内存不足时将无法从`guest`虚拟机回收锁定的内存，这意味着分配大量锁定内存的恶意`guest`虚拟机可能会导致对主机的拒绝服务攻击。因此，除非您的`guest`工作量需要，否则不鼓励使用此选项; 即便如此，强烈建议设置一个 `hard_limit`（参见 内存调整）关于适合特定环境的内存分配，同时减轻上述风险。
    * `source`
        使用该type属性，可以提供`file`以利用文件内存备份或保持默认的“匿名”。从4.10.0开始，您可以选择`memfd`支持。（仅限QEMU/KVM）
    * `access`
        使用该mode属性，指定内存是`shared`还是`private`。这可以通过每个numa节点覆盖`memAccess`。
    * `allocation`
        使用该mode属性，通过提供`immediate`或`ondemand`指定何时分配内存
    * `discard`
        当虚拟机管理程序设置和支持时，内存内容将在`guest`虚拟机关闭之前（或拔出DIMM模块时）被丢弃。请注意，这只是一个优化，并不保证在所有情况下都能正常工作（例如，当虚拟机管理程序崩溃时）。

8. 内存调整

   ```xml
   <domain>
        ...
        <memtune>
            <hard_limit unit='G'>1</hard_limit>
            <soft_limit unit='M'>128</soft_limit>
            <swap_hard_limit unit='G'>2</swap_hard_limit>
            <min_guarantee unit='bytes'>67108864</min_guarantee>
        </memtune>
        ...
    </domain>
   ```

    * `memtune`
        可选`memtune`元素提供有关`domain`的内存可调参数的详细信息。如果省略，则默认为OS提供的默认值。对于QEMU/KVM，这些参数作为一个整体应用于QEMU进程。因此，在计算它们时，需要添加 guest RAM、guest video RAM和QEMU本身的一些内存开销，最后一块很难确定，所以需要猜测并尝试。对于每个可调参数，可以使用与之相同的值来指定输入中数字所在的单位`memory`。为了向后兼容，输出始终为KiB。 `unit`所有`* _limit`参数的可能值范围为0到VIR_DOMAIN_MEMORY_PARAM_UNLIMITED
    * `hard_limit`
        可选`hard_limit`元素是guest虚拟机可以使用的最大内存。该值的单位是`kibibytes`（即1024字节的块）。强烈建议QEMU和KVM的用户不要设置此限制，因为如果猜测太低，域可能被内核杀死，并且确定进程运行所需的内存是一个 不可判定的问题 ; 这么说，如果你已经设置 locked在 内存的支持，因为您的工作负载需求，你就必须考虑到部署的具体情况，并找出一个值`hard_limit` 它足够大，可以支持guest虚拟机的内存要求，但足够小，可以保护主机免受恶意访客锁定所有内存的侵害。
    * `soft_limit`
        可选`soft_limit`元素是在内存争用期间强制执行的内存限制。此值的单位是kibibytes（即1024字节的块）
    * `swap_hard_limit`
        可选`swap_hard_limit`元素是guest虚拟机可以使用的最大内存交换。该值的单位是kibibytes（即1024字节的块）。这必须超过`hard_limit`值
    * `min_guarantee`
        可选`min_guarantee`元素是guest虚拟机的保证最小内存分配。该值的单位是kibibytes（即1024字节的块）。此元素仅受VMware ESX和OpenVZ驱动程序支持。

9. NUMA节点调整

    ```xml
    <domain>
        ...
        <numatune>
            <memory mode="strict" nodeset="1-4,^3"/>
            <memnode cellid="0" mode="strict" nodeset="1"/>
            <memnode cellid="2" mode="preferred" nodeset="2"/>
        </numatune>
        ...
    </domain>
    ```

    * `numatune`
        可选`numatune`元素提供了有关如何通过控制域进程的NUMA策略来调整NUMA主机性能的详细信息。NB，仅由QEMU驱动程序支持。
    * `memory`
        可选`memory`元素指定如何在NUMA主机上为`domain`进程分配内存。它包含几个可选属性。属性mode是`interleave`，`strict`或`preferred`，默认为`strict`。属性`nodeset`使用与`cpuset`元素的属性相同的语法指定NUMA节点vcpu。属性`placement`可以被用于指示domain进程的内存放置模式，它的值可以是`static`或`auto`，默认为`placement`的vcpu，如果`nodeset`已指定,则值是`static`。`auto`表示domain进程只会从查询numa返回的咨询节点集中分配内存，`nodeset`如果指定了属性值，则会忽略该值 。如果`placement`的vcpu是`auto`，并`numatune`没有指定，默认`numatune`使用placement`auto`和mode`strict`将被隐式添加。
    * `momnode`
        可选`memnode`元素可以为每个guest NUMA节点指定内存分配策略。对于那些没有相应memnode元素的节点，memory将使用默认的元素。属性`cellid`为应用了设置的guest虚拟机NUMA节点地址。属性mode并nodeset具有与memory元素相同的含义和语法。此设置与自动放置不兼容。

10. I/O块调整

    ```xml
    <domain>
        ...
        <blkiotune>
            <weight>800</weight>
            <device>
            <path>/dev/sda</path>
            <weight>1000</weight>
            </device>
            <device>
            <path>/dev/sdb</path>
            <weight>500</weight>
            <read_bytes_sec>10000</read_bytes_sec>
            <write_bytes_sec>10000</write_bytes_sec>
            <read_iops_sec>20000</read_iops_sec>
            <write_iops_sec>20000</write_iops_sec>
            </device>
        </blkiotune>
        ...
    </domain>
    ```

    * `blkiotune`
        可选`blkiotune`元素提供了为`domain`调整`Blkio cgroup`可调参数的功能。如果省略，则默认为OS提供的默认值。
    * `weight`
        可选`weight`元素是guest虚拟机的总体I/O权重。该值应在[100,1000]范围内。在内核2.6.39之后，该值可以在[10,1000]的范围内。
    * `device`
        `domain`可以具有多个`device`元素，其进一步调整`domain`使用的每个主机块设备的权重。请注意，如果多个`guest`虚拟机磁盘由同一主机文件系统中的文件支持，则它们可以共享单个主机块设备，这就是为什么此调整参数位于全局`domain`级而不是与每个`guest`虚拟机磁盘设备关联的原因（与此相对应）`iotune` 可以适用于个人的元素`disk`）。每个`device`元素都有两个必需的子元素，`path`描述了设备的绝对路径，以及`weight`给出该设备的相对权重，范围为[100,1000]。在内核2.6.39之后，该值可以在[10,1000]的范围内。
      * `read_bytes_sec`
        读取吞吐量限制，以每秒字节为单位
      * `write_bytes_sec`
        写吞吐量限制，以每秒字节数为单位
      * `read_iops_sec`
        读取每秒I/O操作限制。
      * `write_iops_sec`
        每秒写入I/O操作限制。

11. 资源分区

    ```xml
    ...
    <resource>
    <partition>/virtualmachines/production</partition>
    </resource>
    ...
    ```

    * 管理程序可以允许将虚拟机放置到资源分区中，可能嵌套所述分区。该`resource`元素将与资源分区相关的配置组合在一起。它当前支持子元素，`partition`其内容定义了放置域的资源分区的绝对路径。如果未列出任何分区，则域将放置在默认分区中。app/admin负责确保在启动guest虚拟机之前存在分区。默认情况下，只能假定存在（特定于虚拟机管理程序）默认分区。
    * QEMU和LXC驱动程序当前支持资源分区，这些驱动程序将分区路径映射到所有已安装控制器中的cgroups目录。

12. CPU模型和拓扑
    > 可以这里指定CPU模型，其功能和拓扑的要求。
    > 参考<https://libvirt.org/formatdomain.html#elementsCPU>

13. 事件配置
    有时需要覆盖对各种事件采取的默认操作。并非所有虚拟机管理程序都支持所有事件和操作。这些动作可以作为`libvirt`API:`virDomainReboot`， `virDomainShutdown`或 `virDomainShutdownFlags`的结果。使用`virsh reboot`或`virsh shutdown`也会触发事件。

    ```xml
    ...
    <on_poweroff>destroy</on_poweroff>
    <on_reboot>restart</on_reboot>
    <on_crash>restart</on_crash>
    <on_lockfailure>poweroff</on_lockfailure>
    ...
    ```

    * 以下元素集合允许在guest虚拟机操作系统在触发生命周期操作时执行指定的操作。一个常见的用例是在执行初始操作系统安装时强制重启被视为断电。这允许为第一次安装后启动重新配置VM。
      * `on_poweroff`
          此元素的内容指定`guest`虚拟机请求断电时要采取的操作。

      * `on_reboot`
          此元素的内容指定`guest`虚拟机请求重新引导时要执行的操作。

      * `on_crash`
          此元素的内容指定`guest`虚拟机崩溃时要执行的操作。

      * 以上三个元素有以下属性
        * `destroy`
            `domain`将终止并释放所有资源
        * `restart`
            `domain`将终止，然后使用相同的配置重新启动
        * `preserve`
            `domain`将被终止并保留资源以允许分析
        * `rename-restart`
            `domain`将终止，然后使用新名称重新启动
      * QEMU/KVM支持`on_poweroff`和`on_reboot`事件处理`destroy`和`restart`动作。`on_reboot`事件的`preserve`操作被视为`destroy`，`on_poweroff`事件的`rename-restart`操作被视为`restart`。

      * `on_crash`事件额外支持以下操作
        * `coredump-destroy`
            `domain`崩溃后，`core`将被转储，然后`domain`将被完全终止并释放所有资源
        * `coredump-restart`
            `domain`崩溃后，`core`将被转储，然后将使用相同的配置重新启动`domain`

    * `on_lockfailure`配置锁管理器失去资源锁应该采取什么行动。libvirt可识别以下操作，但并非所有操作都需要由各个锁管理器支持。如果未指定任何操作，则每个锁定管理器将采用其默认操作。
      * poweroff
        域名将被强制关闭。
      * restart
        域将关闭并再次启动以重新获取其锁定。
      * pause
        域将被暂停，以便在解决锁定问题时手动恢复。
      * ignore
        保持域运行，好像什么也没发生。

14. 电源管理
    > 可以强制启用或禁用对客户OS的BIOS通告。（注意：只有qemu驱动程序支持）

    ```xml
    ...
    <pm>
    <suspend-to-disk enabled='no'/>
    <suspend-to-mem enabled='yes'/>
    </pm>
    ...
    ```

    * `pm`
        `S3(suspend-to-mem)`和`S4(suspend-to-disk)`这些元素启用`yes`或禁用`no`对BIOS的ACPI睡眠状态支持。如果未指定任何内容，则管理程序将保留其默认值。
        注意：此设置无法阻止客户操作系统执行挂起，因为客户操作系统本身可以选择避免睡眠状态的不可用（例如，通过完全关闭S4）。

15. 管理程序功能
    > 管理程序可以允许打开/关闭某些 `CPU/machine` 功能。

    ```xml
    ...
    <features>
        <pae/>
        <acpi/>
        <apic/>
        <hap/>
        <privnet/>
        <hyperv>
            <relaxed state='on'/>
            <vapic state='on'/>
            <spinlocks state='on' retries='4096'/>
            <vpindex state='on'/>
            <runtime state='on'/>
            <synic state='on'/>
            <reset state='on'/>
            <vendor_id state='on' value='KVM Hv'/>
            <frequencies state='on'/>
            <reenlightenment state='on'/>
            <tlbflush state='on'/>
            <ipi state='on'/>
            <evmcs state='on'/>
        </hyperv>
        <kvm>
            <hidden state='on'/>
        </kvm>
        <pvspinlock state='on'/>
        <gic version='2'/>
        <ioapic driver='qemu'/>
        <hpt resizing='required'>
            <maxpagesize unit='MiB'>16</maxpagesize>
        </hpt>
        <vmcoreinfo state='on'/>
        <smm state='on'>
            <tseg unit='MiB'>48</tseg>
        </smm>
        <htm state='on'/>
    </features>
    ...
    ```

    `features`元素中列出了所有要素，省略了可切换的要素标记将其关闭。可以通过[`capabilities XML`](https://libvirt.org/formatcaps.html)和[domain capabilities XML](https://libvirt.org/formatdomaincaps.html)来找到可用的功能，但完全虚拟化域的通用集合是：
    * `pae`
        物理地址扩展模式允许32位客户机处理超过4GB的内存。
    * `acpi`
        ACPI对于电源管理非常有用，例如，对于正常关闭，`KVM guest`需要它才能正常工作。
    * `apic`
        APIC允许使用可编程IRQ管理. 有一个可选的属性`eoi`与值`on`和`off`切换EOI的用于客户的可用性（中断结束）
    * `hap`
        根据不同的`state`属性(值on， off)启用或禁用使用硬件辅助寻址。默认情况下，如果管理程序检测到硬件辅助分页的可用性的花值为`on`。
    * `viridian`
        为半虚拟化来宾操作系统启用Viridian管理程序扩展
    * `privnet`
        始终创建专用网络命名空间。如果定义了任何接口设备，则会自动设置。此功能仅与基于容器的虚拟化驱动程序（如LXC）相关。
    * `hyperv`
        启用各种功能可改善运行Microsoft Windows的guest虚拟机的行为。
        | 特征            | 描述                                         | 值                                |
        | --------------- | -------------------------------------------- | --------------------------------- |
        | relaxed         | 放松对计时器的限制                           | on,off                            |
        | vapic           | 启用虚拟APIC                                 | no,off                            |
        | spinlocks       | 启用spinlocks支持                            | no,off                            |
        | runtime         | 处理访客代码和代表访客代码所花费的处理器时间     | no,off                            |
        | synic           | 启用合成中断控制器（SyNIC）                  | no,off                            |
        | stimer          | 启用S​​yNIC计时器                            | no,off                            |
        | reset           | 启用管理程序重置                             | no,off                            |
        | vendor_id       | 设置管理程序供应商ID                         | no,off;value-字符串，最多12个字符 |
        | frequencies     | 暴露频率MSR                                  | no,off                            |
        | reenlightenment | 启用迁移的重新启动通知                       | no,off                            |
        | tlbflush        | 启用PV TLB刷新支持                           | no,off                            |
        | ipi             | 启用PV IPI支持                               | no,off                            |
        | evmcs           | 启用Enlightened VMCS                         | no,off                            |
    * `pvspinlock`
        通过暴露`pvticketlocks`机制通知`guest`虚拟机主机支持`paravirtual spinlocks`。可以使用`state='off'` 属性显式禁用此功能。
    * `kvm`
        用于更改KVM管理程序行为的各种功能。
        * `hidden`
            隐藏基于标准MSR发现的KVM管理程序
    * `pmu`
        根据`state`属性(值`on`，`off`，default `on`)启用或禁用guest虚拟机的性能监视单元。
    * `vmport`
        根据状态属性（值为`on`、`off`、default `on`）启用或禁用对`vmware IO端口`、`vmmouse`等的模拟。
    * `gic`
        启用使用通用中断控制器而不是APIC的体系结构，以便处理中断。例如，`aarch64`体系结构使用`gic`而不是`apic`。可选属性`version`指定了`gic`版本；但是，并非所有管理程序都支持它。接受值为`2`、`3`和`主机`。
    * `smm`
        根据`state`属性（值`on`， `off`默认值`on`）启用或禁用系统管理模式。
    * `ioapic`
    * `hpt`
    * `vmcoreinfo`
    * `htm`
    * `nested-hv`
16. 保持时间
17. 绩效监测事件
18. 设备
19. Vsock
20. 安全标签
21. 钥匙包裹
22. 启动安全性

23. 我的xml配置

    ```xml
    <domain type='kvm'>
        <name>node1</name>
        <title>This is my first test kvm</title>
        <description>我是个描述</description>
        <memory unit='MB'>2048</memory>
        <currentMemory unit='MB'>1024</currentMemory>
        <vcpu placement='static' cpuset="0" current="1">2</vcpu>
        <cpu>
            <topology sockets='1'  cores='1' threads='2'/>
        </cpu>
        <cpu mode='host-model'>
            <model fallback='allow'/>
        </cpu>
        <os>
            <type arch='x86_64'>hvm</type>
            <bootmenu enable='yes'/>
        </os>

        <features>
            <acpi/>
            <apic/>
            <pae/>
            <!-- `hyperv`选项优化windows的，linux可以不用 -->
            <hyperv>
                <relaxed state='on'/>
                <vapic state='on'/>
                <spinlocks state='on' retries='4096'/>
                <vpindex state='on'/>
                <runtime state='on'/>
                <synic state='on'/>
                <reset state='on'/>
                <vendor_id state='on' value='KVM Hv'/>
                <frequencies state='on'/>
                <reenlightenment state='on'/>
                <tlbflush state='on'/>
                <ipi state='on'/>
                <evmcs state='on'/>
            </hyperv>
            <viridian/>
        </features>

        <clock offset='localtime'/>
        <on_poweroff>destroy</on_poweroff>
        <on_reboot>restart</on_reboot>
        <on_crash>restart</on_crash>
        <on_lockfailure>restart</on_lockfailure>

        <devices>
            <emulator>/usr/libexec/qemu-kvm</emulator>

            <!-- 使用磁盘文件 -->
            <disk type='file' device='disk'>
                <driver name='qemu' type='qcow2' cache='writeback' io='threads'/>
                <source file='/home/data/kvm/node1.0.qcow2'/>
                <boot order='1'/>
                <target dev='vda' bus='virtio'/>
            </disk>

            <!-- 使用存储池中的卷 -->
            <disk type='volume' device='disk'>
                <driver name='qemu' type='qcow2' cache='writeback' io='threads'/>
                <source pool='kvm-host' volume='node1'/>
                <target dev='vdb' bus='virtio'/>
            </disk>

            <!-- 加载光盘镜像 -->
            <disk type='file' device='cdrom'>
                <source file='/home/cc/hypervisor/images/ubuntu-18.04.2-live-server-amd64.iso'/>
                <target dev='hdc'/>
                <boot order='2'/>
                <readonly/>
            </disk>

            <serial type='pty'>
                <target port='0'/>
            </serial>

            <console type='pty'>
                <target port='0'/>
            </console>

            <hostdev mode='subsystem' type='pci' managed='yes'>
            </hostdev>

            <interface type='bridge'>
                <source bridge='birbr0'/>
                <model type='virtio'/>
                <driver name='vhost' queue='4'>
            </interface>

            <input type='mouse' bus='virtio'/>
            <input type='keyboard' bus='virtio'/>
            <input type='tablet' bus='virtio'/>

            <graphics type='vnc' port='5900' autoport='yes' listen='0.0.0.0' keymap='en-us'>
                <listen type='address' address='0.0.0.0'/>
            </graphics>

            <graphics  type='spice' port='4000' autoport='no' listen='0.0.0.0'>
                <listen  type='address' address='0.0.0.0'/>
                <agent_mouse  mode='off'/>
            </graphics>

            <!-- 随机数优化 -->
            <rng model='virtio'>
                <backend model='random'>/dev/random</backend>
            </rng>

        </devices>
    </domain>
    ```

## 管理工具

> 这里直接写命令的常用方式，涉及到的参数的意义，可从[优化和介绍kvm里](#优化和介绍KVM)详细查看
> 默认的KVM虚拟机工作目录为`/var/lib/libvirt`，配置文件目录`/etc/libvirt/qemu`,主机名称+xml结尾的文件即其相关虚拟机的配置文件，需要修改其配置，也可以直接修改xml文件实现（不建议）。其中autostart目录定义的配置文件会随主机一起启动，而network定义了虚拟机使用桥接网络时的网关网卡的相关配置。

### qemu-img

> 创建镜像文件，参考[磁盘存储](#磁盘存储)

```sh
qemu-img create -f qcow2 node1.qcow2 10G  # 创建大小为10G，qcow2格式的镜像文件
qemu-img info node1.qcow2                 # 查看镜像信息
qemu-img convert -f raw -o qcow2 node1.img node1.qcow2     # 更改镜像格式
```

### virsh

> virsh 是一个用于监控系统程序和客户机虚拟机器的命令行接口（CLI）工具。virsh 命令行工具建立在 libvirt 管理 API，并作为可选择的一个运行方式来替代 qemu-kvm 命令和图形界面的 virt-manager 应用。无特权的用户以只读的方式使用 virsh 命令；有根用户权限的用户可以使用该命令的所有功能。virsh 是一个对虚拟环境的管理任务进行脚本化的理想工具。另外，virsh 工具是 guest 操作系统 域的一个主要管理接口，可以用于创造、暂停和关闭“域”，或罗列现有域。这一工具作为 libvirt-client 软件包中的一部分被安装。
>
> KVM虚拟机依靠两个主要文件来启动，一个是img文件，一个是xml配置文件

* 安装

    ```sh
    dnf install libvirt-client
    or
    yum install libvirt-client
    ```

* 命令常用方式

    ```sh
    virsh define node1.xml          # 导入虚拟机配置（xml格式）（这个虚拟机还不是活动的)
    virsh create node1.xml          # 创建虚拟机（创建后，虚拟机立即执行，成为活动主机）
    virsh start node1               # 开启node1虚拟机
    virsh suspend node1             # 暂停虚拟机（不释放资源）
    virsh resume node1              # 启动暂停的虚拟机
    virsh shutdown node1            # 正常关闭虚拟机
    virsh destroy node1             # 强制关闭虚拟机
    virsh dominfo node1             # 查看虚拟机的基本信息
    virsh domname 2                 # 查看序号为2的虚拟机别名
    virsh domid node1               # 查看虚拟机的序号
    virsh domstate node1            # 查看虚拟机的当前状态
    virsh list --all                # 查看所有虚拟机
    virsh domstate node1            # 查看虚拟机的xml配置（可能和定义虚拟机时的配置不同，因为当虚拟机启动时，需要给虚拟机分配id号、uuid、vnc端口号等等）
    virsh setmem node1 512000       # 给不活动的虚拟机设置内存大小
    virsh setvcpus node1 4          # 给不活动的虚拟机设置CPU个数
    virsh destroy node1             # 销毁虚拟机，不删除虚拟机配置
    virsh undefine node1            # 删除虚拟机配置
    virsh dumpxml node1             # 显示虚拟机的xml配置
    virsh edit node1                # 编辑xml配置文件
    virsh vncdisplay node1          # 获取虚拟机的vnc连接端口
    ```

* 存储池来管理存储

    ```sh
    virsh pool-define-as kvm_images dir - - - - "/home/data/kvm/pool" # 定义存储池
    virsh pool-build kvm_images     # 建立基于文件夹的存储池
    virsh pool-start kvm_images     # 使存储池生效
    virsh pool-info kvm_images      # 查看存储池详情
    virsh pool-list --all           # 查看所有存储池

    # 创建完存储池后，就可以通过创建卷的方法来创建虚拟机的磁盘
    virsh vol-create-as kvm_images node1.qcow2 10G --format qcow2   # 创建卷
    virsh pool-refresh kvm_images   # 刷新存储池
    virsh vol-info kvm_images       # 查看存储池里边的存储卷信息
    virsh vol-info node1.qcow2 kvm_images       # 查看存储池里边单独一个卷的信息
    virsh vol-dumpxml node1.qcow2 kvm_images    # 查看存储池里边的一个卷的详细信息
    ```

* 虚拟机备份

    ```sh
    virsh save --bypass-cache node1 /var/lib/libvirt/save/node1_1.save --running    # 备份
    virsh restore /var/lib/libvirt/save/node1_1.save --bypass-cache --running       # 还原
    ```

* 虚拟机快照

    > 如果要使用kvm的快照功能，就必须使用qcow2的磁盘格式，而raw只支持内存快照，如果不是，请修改

    ```sh
    virsh snapshot-create node1 node1.snap1 # 创建快照
    virsh snapshot-revert node1 node1.snap1 # 恢复快照
    virsh snapshot-list node1               # 查看快照
    virsh snapshot-delete node1 node1.snap1 # 删除快照
    ```

* 虚拟机迁移

    > KVM虚拟机依靠两个主要文件来启动，一个是img文件，一个是xml配置文件,因此迁移的时候，可以直接迁移这两个文件就能实现静态迁移。如果img文件存放在共享存储，则更为方便，只用迁移xml配置文件，就可以实现静态迁移。
    > 当然，virsh命令也可以迁移虚拟机，不过要求目标主机与当前主机的应用环境须保持一致，其命令格式如下：

    ```sh
    virsh migrate --live node1 qemu+tcp//destnationip/system tcp://destnationip
    ```

* 通过xml创建虚拟机，xml写法请参考[xml](#libvirt配置文件)


### virt-viewer

> 远程连接工具，支持`vnc`和`spice`


### 其他工具

请查看redhat官方介绍[其他工具](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/virtualization_getting_started_guide/sect-virtualization_getting_started-tools-other)

## 磁盘&存储配置

1. 存储池和储存卷

    > “存储池 ”（storage pool）即一个由 “libvirt” 管理的文件、目录或储存设备，其目的是为虚拟机提供储存空间。存储池被分隔为存储 “卷 ”（volume），可以用来存储虚拟机镜像或附加到虚拟机作为额外额存储。多个客机可共享同一储存池，允许储存资源得到更好分配
    >
    >储存池进一步划分为“储存卷 ”（storage volume）。储存卷是物理分区、LVM 逻辑卷、基于文件的磁盘镜像及其它由 libvirt 控制的储存形式的抽象层。不论基于何种硬件，储存卷会作为本地储存设备呈现给虚拟机。
    >
    > 个人理解为，存储池就是从本地硬盘或网络硬盘划分出来一个区域给虚拟机 guest 系统使用的存储空间，每个卷相对guest系统来说就是一块硬盘

    * 本地存储池

        > 本地储存池直接连接到主机服务器。它们包括本地目录、直接连接的磁盘、物理分区和本地设备上的 LVM 卷组，因为本地存储池不支持实时迁移，所以它可能不适用于某些生产环境。

    * 网络存储池

        > 网络储存池包括在网络上使用标准协议共享的储存设备。使用 virt-manager 在主机间进行虚拟机的迁移需要网络储存，但是当使用 virsh 迁移时，它是可选的。网络储存池由 libvirt 进行管理。

2. 主机存储

    > 磁盘镜像可以储存在一系列和主机相连的本地或远程存储中。(可以存储在本地或者网络磁盘中)
    > KVM虚拟机的磁盘镜像从存储方式来看分为两种：存储于文件系统，直接使用使用裸设备。

    * 镜像文件(存储在文件系统中)
        > 镜像文件储存在主机文件系统中。它可以储存在本地文件系统中，如 ext4 或 xfs；或网络文件系统中，如 NFS
        > 创建一个镜像文件给 guest 系统当作磁盘使用

        例如 libguestfs 这样的工具，能管理、备份及监控文件。

        KVM 上的磁盘镜像格式包括：
        * raw

            > 当对磁盘 I/O 性能要求非常高，而且通常不需要通过网络传输镜像文件时，可以使用 raw 文件。**不推荐使用**

            raw 镜像文件指不包含附加元数据的磁盘内容。

            假如主机文件系统允许，raw 文件可以是预分配（pre-allocated）或稀疏（sparse）。稀疏文件根据需求分配主机磁盘空间（动态存储)，因此它是一种精简配置形式（thin provisioning）。预分配文件的所有空间需要被预先分配，但它比稀疏文件性能好。

        * qcow2

            > Red Hat Enterprise Linux 7.0 及更新版本支持 qcow2 v3 镜像文件格式。
            > 动态存储，用多少就实际占用多少物理存储空间

            qcow2 镜像文件提供许多高级磁盘镜像特征，如快照、压缩及加密。它们可以用来代表通过模板镜像创建的虚拟机。因为只有虚拟机写入的扇区部分才会分配在镜像中，所以 qcow2 文件的网络传输效率较高。

    * lvm卷(直接使用裸设备)

        > LVM 精简配置为 LVM 卷提供快照和高效的空间使用，它可以作为 qcow2 的一种替代选择。
        > 创建一个lvm卷给 guest 系统当作硬盘使用

        逻辑卷可用于磁盘镜像，并使用系统的 LVM 工具进行管理。 由于它使用更简单的块储存模式，LVM 比文件系统的性能更高。

    * 主机设备(直接使用裸设备)

        > 在 SAN 而不是主机上进行储存管理时，可以使用主机设备

        主机设备如物理 CD-ROM、原始磁盘或 LUN 都可以提供给客机。这使得 SAN 或 iSCSI LUN 还有本地 CD-ROM 都可以提供给客机所用。

    * 分布式存储系统(直接使用裸设备)

        > Gluster 卷可用作磁盘镜像。它提供了高效的、使用网络的集群存储。

        Red Hat Enterprise Linux 7 包括在 GlusterFS 上对磁盘镜像的原生支援。这使 KVM 主机可以直接从 GlusterFS 卷引导虚拟机镜像，并使用 GlusterFS 卷中的镜像作为虚拟机的数据磁盘。与 GlusterFS FUSE 相比，KVM 原生支持性能更好。

## KVM网络互联配置

## 优化KVM

### CPU优化

* 要考虑CPU的数量问题，所有guest cpu的总数目不要超过物理机CPU总数目，如果超过，则将对性能带来严重影响，建议选择复制主机CPU配置。

* “CPU 型号 ”（CPU model）规定了哪些主机 CPU 功能对客机操作系统有效。 qemu-kvm 和 libvirt 包含了几种当前处理器型号的定义，允许用户启用仅在新型 CPU 型号中可用的 CPU 功能。 对客机有效的的 CPU 功能取决于主机 CPU 的支持、内核以及 qemu-kvm 代码。

* 为了使虚拟机可以在具有不同 CPU 功能集的主机间安全地进行迁移，qemu-kvm 在默认状态下不会把主机 CPU 的所有功能都提供给客机操作系统，而是根据所选的 CPU 型号来为虚拟机提供相关的 CPU 功能。如果虚拟机启用了某个 CPU 功能，则此虚拟机无法迁移到不支持向客机提供此功能的主机上。

### 内存优化

> 主要分为主机优化和guest优化

1. KSM（kernel Samepage Merging）相同页合并

    > 内存分配的最小单位是page（页面），默认大小是4KB，可以将host主机内容相同的内存合并，以节省内存的使用。
    >
    > 当KVM上运行许多相同系统的客户机时，客户机之间将有很多内存页是完全相同的，特别是只读的内核代码页完全可以在客户机之间共享，从而减少客户机占用的内存资源，也能同时运行更多的客户机。
    >
    > 使用KSM存在性能损失，在一般的环境中，性能损失大概是10%，这也是在某些环境中关闭KSM的原因。
    >
    > 建议开启KSM的同时不要使用memory balloon，两种内存优化方案会降低系统的性能;
  
    * 在什么时候开启KSM？

      * 如果目标是运行尽可能多的虚拟机，而且性能不是问题，应该保持KSM处于运行状态。例如KSM允许运行30个虚拟机的主机上运行40个虚拟机，这意味着最大化硬件使用效率。

      * 如果服务器在运行相对较少的虚拟机并且性能是个问题时，那么应该关闭KSM。

      * 如果CPU是整个宿主机的资源瓶颈则不建议使用KSM，因为KSM会带来相应的CPU开销
  
    * KSM文件目录：

        ```bash
        $ ls /sys/kernel/mm/ksm/
        总用量 0
        -r--r--r--. 1 root root 4096 11月 30 17:05 full_scans           # 对可合并的内存区域扫描过的次数
        -rw-r--r--. 1 root root 4096 11月 30 17:05 max_page_sharing
        -rw-r--r--. 1 root root 4096 11月 30 17:05 merge_across_nodes   # 是否可以合并来自不同NUMA节点的页面。参数更改0,避免跨NUMA节点合并页面
        -r--r--r--. 1 root root 4096 11月 30 17:05 pages_shared         # 记录合并后共有多少内存页
        -r--r--r--. 1 root root 4096 11月 30 17:05 pages_sharing        # 记录有多少内存页正在使用被合并的共享页，不包括合并的内存页本身
        -rw-r--r--. 1 root root 4096 11月 30 17:05 pages_to_scan        # 决定每次扫描多少个页面默认是100，越大越好，超过2000无效，需要开启两个服务ksmtuned和tuned,支持更多页面
        -r--r--r--. 1 root root 4096 11月 30 17:05 pages_unshared       # 因为没有重复内容而不能被合并的内存页数量
        -r--r--r--. 1 root root 4096 11月 30 17:05 pages_volatile       # 因为内容很容易变化而不能被合并的内存页数量
        -rw-r--r--. 1 root root 4096 11月 30 17:05 run                  # 查看是否开启KSM，0为关闭，1为开启,临时开启 echo 1 > run 2为停止ksmd并分离已合并的内存页
        -rw-r--r--. 1 root root 4096 11月 30 17:05 sleep_millisecs      # 决定多长时间扫描一次
        -r--r--r--. 1 root root 4096 11月 30 17:05 stable_node_chains
        -rw-r--r--. 1 root root 4096 11月 30 17:05 stable_node_chains_prune_millisecs
        -r--r--r--. 1 root root 4096 11月 30 17:05 stable_node_dups
        -rw-r--磁盘r--. 1 root root 4096 11月 30 17:05 use_zero_pages
        ```

    * KSM管理
     ksm可以直接配置/sys/kernel/mm/ksm/目录下的文件
  
    * ksmtuned管理
        > ksmtuned会一直保持循环执行，以调节ksm服务运行。

        * 安装ksmtuned管理工具

            ```sh
            yum install ksmtuned
            ```

        * 编辑ksmtuned配置文件

            ```ini
            # ksm每次内存扫描的时间;
            KSM_MONITOR_INTERVAL = 60

            # 表示每次扫描休息的间隔时间(最小值为10)，KSM扫描会占用一些CPU的开销，所以当KVM虚拟机数量或者应用软件较少时可以调整KSM_SLEEP_MSEC至一个较大的值，反之则设置较小的值;同时当Hypervisor里面的虚拟机的内存调优到达一个稳定状态，也可以根据情况把这个参数调小节省CPU的开销;
            KSM_SLEEP_MSEC = 10

            # 内存页合并增加数量;
            KSM_NPAGES_BOOST = 300

            # 内存页合并减少数量;
            KSM_NPAGES_DECAY = -50

            # 内存页合并最小值;
            KSM_NPAGES_MIN = 64

            # 内存页合并最大值
            KSM_NPAGES_MAX = 1250

            # 临界值系数，越大合并的越多
            KSM_THRES_COEF = 20

            # 临界值常量
            KSM_THRES_CONST = 2048

            # 取消注释以下内容以启用ksmtuned调试信息
            LOGFILE = /var/log/ksmtuned
            # DEBUG = 1
            ```

        * ksmtuned工作方法：
            * ksm先取得到宿主机的总内存“total”， thres = total * KSM_THRES_COEF / 100。然后与临界值常量KSM_THRES_CONST进行比较，如果thres 小于 KSM_THRES_CONST，那么thres就 等于 KSM_THRES_CONST;

            * 计算qemu进程使用的内存总和：committed; 当且仅当committed + thres > total时，才开始操作内存页合并。所以调整临界值常量与临界值系数可以确定临界值thres, 从而有效地调整ksm工作方式。

            * 判断剩余内存量free与thres的大小，如果free < thres，ksm每次扫描后内存页合并数会增加 KSM_NPAGES_BOOST，该参数上限为KSM_NPAGES_MAX;反之如果free>thres内存页合并数量会 减少 KSM_NPAGES_DECAY，下限为KSM_NPAGES_MIN。
  
2. 对内存设置限制
    > 如果我们有多个虚拟机，为了防止某个虚拟机无节制的使用内存资源，导致其他虚拟机无法正常使用，就需要对使用的内存进行限制。
    >
    > 不同的管理工具不同的配置方法，以virs为例

    * 查看当前虚拟机的内存限制，单位为KB

    ```sh
    virsh memtune c7-1

    hard_limit     : 无限制       # 强制最大内存
    soft_limit     : 无限制       # 可用最大内存
    swap_hard_limit: 无限制       # 强制最大swap使用大小
    ```

    * 设置强制最大内存为100MB，在线生效。

    ```sh
    virsh memtune c7-1 --hard-limit 1024000 --live

    virsh memtune c7-1          # 查看
    hard_limit     : 1024000
    soft_limit     : 无限制
    swap_hard_limit: 无限制
    ```

3. 大页后端内存

    > 在逻辑地址想物理地址转换时，CPU保持一个翻译后备缓冲器TLB，用来缓冲转换结果，而TLB容量很小，所以如果page很小，TLB很容易就充满，这样就容易导致cache miss，相反page变大，TLB需要保存的缓存项就变少，就会减少cache miss，通过为客户端提供大页后端内存，就能减少客户机消耗的内存并提高TLB命中率，从而提高KVM性能。
    >
    > 在RHEL里，大页的大小可以是2M,1G.默认情况下，已经开启了透明大页功能
    >
    > 配置大页面后，系统在开机启动时会首选尝试在内存中找到并预留连续的大小为 HugePages_Total * Hugepagesize 的内存空间。如果内存空间不满足，则启动会报错 Kernel Panic, Out of Memory 等错误。
    > 如只配置了一个大小的大页面，可以通过 /proc/meminfo 中的 Hugepagesize 和 HugePages_Total 计算出大页面所在内存空间的大小。这部分空间会被算到已用的内存空间里，即使还未真正被使用
  
    * 查看内存信息，无可用大页

        ```sh
        cat /proc/meminfo | grep Huge
        HugePages_Total:       0        # 大叶面的数量
        HugePages_Free:        0        # 未使用的大叶面数量
        HugePages_Rsvd:        0
        HugePages_Surp:        0
        Hugepagesize:       2048 kB     # 每个大页面的大小
        ```

    * 查看设置的大页面的数目

        ```bash
        cat /proc/sys/vm/nr_hugepages
        ```

    * 指定大页需要的内存页面数量,页面大小一般不更改，

        ```bash
        echo 25000 > /proc/sys/vm/nr_hugepages  # 临时生效
        sysctl -w vm.nr_hugepages=25000         # 永久生效
        或者
        vim /etc/sysctl.conf
        vm.nr_hugepages=25000
        sysctl -p                               # 重新加载配置
        ```

4. 内存气泡

5. 页面交换
    qemu-kvm虚拟机作为系统的进程存在，所使用的内存可以利用系统的swap能力进行换出。不需要特别设置。
6. 页面压缩
    未见支持。

### I/O等设备使用半虚拟化设备

1. 采用virtio磁盘控制器

    > kvm设计了virtio类型的磁盘控制器，是针对磁盘和网络的一个半虚拟化接口，以提高效率为目的。
    > guest 系统需要安装半虚拟化驱动，Linux内核中已经集成进去了，window平台的话，必须手动安装

    * virtio-scsi
        半虚拟化 SCSI 控制器设备是一种更为灵活且可扩展的 virtio-blk 替代品，irtio-scsi 客机能继承目标设备的各种特征，并且能操作几百个设备，相比之下，virtio-blk 仅能处理 28 台设备。使用大量磁盘或高级储功能（如 TRIM）的客机推荐使用的半虚拟化设备。

    * virtio-blk
        适用于向客机提供镜像文件的半虚拟化储存设备。virtio-blk 可以为虚拟机提供最好的磁盘 I/O 性能，但比 virtio-scsi 的功能少。

2. 其的半虚拟化设备

    * virtio-net
        半虚拟化网络设备是虚拟化网络设备，它为虚拟机提供了网络访问能力，并可以提供网络性能及减少网络延迟。

    * 半虚拟化时钟
        使用时间戳计数器（TSC，Time Stamp Counter）作为时钟源的客机可能会出现与时间相关的问题。KVM 在主机外围工作，这些主机在向客机提供半虚拟化时间时没有固定的 TSC。此外，半虚拟化时钟会在客机运行 S3 或挂起 RAM 时帮助调整所需时间。半虚拟化时钟不支持 Windows guest。

    * virtio-serial
        半虚拟化串口设备是面向比特流的字符流设备，它为主机用户空间与客机用户空间之间提供了一个简单的交流接口。

    * virtio-balloon
        > 建议开启KSM的同时不要使用memory balloon，两种内存优化方案会降低系统的性能;
        > 个人理解，动态使用内存，虚拟机使用多少就占用多少内存，但是也不可以超过限定值

        气球（ballon）设备可以指定虚拟机的部分内存为没有被使用（这个过程被称为气球“充气 ” — inflation），从而使这部分内存可以被主机（或主机上的其它虚拟机）使用。当虚拟机需要这部分内存时，气球可以进行“放气 ”（deflated），主机就会把这部分内存重新分配给虚拟机。

    * virtio-rng
        半虚拟化随机数字生成器使虚拟机可以直接从主机收集熵或随意值来使用，以进行数据加密和安全。因为典型的输入数据（如硬件使用情况）不可用，虚拟机常常急需熵。取得熵很耗时；virtio-rng 通过直接把熵从主机注入客机虚拟机从而使这个过程加快 。
    * QXL
        半虚拟化图形卡与 QXL 驱动一同提供了一个有效地显示来自远程主机的虚拟机图形界面。SPICE 需要 QXL 驱动。

3. 直接使用物理主机设备

    > 特定硬件平台允许虚拟机直接访问多种硬件设备及组件。在虚拟化中，此操作被称为 “设备分配 ”（device assignment）。设备分配又被称作 “传递 ”（passthrough）。
    > 需要物理设备支持 device assignment

    * VFIO 设备分配

        虚拟功能 I/O（VFIO）把主机系统上的 PCI 设备与虚拟机直接相连，允许客机在执行特定任务时有独自访问 PCI 设备的权限。这就象 PCI 设备物理地连接到客机虚拟机上一样。通过把设备分配从 KVM 虚拟机监控系统中(半虚拟化和全虚拟化)移出，并在内核级中强制进行不同guest之间进行设备隔离，VFIO 安全性更高且与安全启动兼容。在 Red Hat Enterprise Linux 7 中，它是默认的设备分配机制。VFIO 可以分配的设备数量为32个， 并且支持对 NVIDIA GPU 的分配。

    * USB传递

        > USB 设备分配允许客机拥有在执行特定任务时有专有访问 USB 设备的权利。这就象 USB 设备物理地连接到虚拟机上一样。

    * SR-IOV
        > 简单的说，SR-IOV是一种虚拟化方案，用来使一个PCIe的物理设备，能虚拟出多个设备，这些虚拟出来的设备看起来就像是多个真实的物理设备一样，并获得能够与本机性能媲美的 I/O 性能。
        >
        > SR-IOV现在最多是用在网卡上，kvm虚拟机的网卡功能一般会下降到实体机的30-50%，如果改用SR-IOV会大大提高网卡性能。

      * SR-IOV 有2种功能：
        * 物理功能 (Physical Function, PF):
          就是标准的PCIe的功能了
        * 虚拟功能 (Virtual Function, VF):
          与物理功能关联的一种功能。VF 是一种轻量级 PCIe 功能，可以与物理功能以及与同一物理功能关联的其他 VF 共享一个或多个物理资源。VF 仅允许拥有用于其自身行为的配置资源。

    * NPIV

        > NPIV 可以提供带有企业级存储解决方案的高密度虚拟环境。

        N_Port ID Virtualization（NPIV）是对光纤通道设备有效的功能。NPIV 共享单一物理 N_Port 作为多个 N_Port ID。NPIV 为 HBA（光纤通道主机总线适配器，Fibre Channel Host Bus Adapter）提供和 SR-IOV 为 PCIe 接口提供的功能相似的功能。有了 NPIV，可以为 SAN（存储区域网络，Storage Area Network）提供带有虚拟光纤通道发起程序的虚拟机。
