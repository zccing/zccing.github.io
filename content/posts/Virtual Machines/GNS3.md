---
title: "linux安装GNS3的VPCS、dynamips、IOU支持"
Date: 2020-08-30T01:37:56+08:00
draft: false
keywords: ['git', '安装git','编译安装git','linux']
description: "使用编译的方式在linux服务器上安装git"
tags: ['GNS3']
categories: ['Virtual Machines']
---

linux安装GNS3的VPCS、dynamips、IOU支持，目前是使用fedora29来安装的，可能也支持fedora30，但是fedora31未经过测试，不晓得支持不
<!--more-->

# gns3 安装使用

## 1 gns3 vpcs and dynamips 安装

```bas
dnf copr enable athmane/gns3-extra
dnf install vpcs dynamips
```

## 2 gns3 IOU for linux 安装

* `fedora 29`

    ```bash
    sudo dnf install git bison flex gcc make openssl-libs.i686 libgcc.i686
    git clone http://github.com/ndevilla/iniparser.git
    cd iniparser
    make
    sudo cp libiniparser.* /usr/lib/
    sudo cp src/iniparser.h /usr/local/include
    sudo cp src/dictionary.h /usr/local/include
    cd ..

    git clone https://github.com/GNS3/iouyap.git
    cd iouyap
    make
    sudo make install
    ```
