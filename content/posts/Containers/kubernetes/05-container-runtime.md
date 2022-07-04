---
title: "k8s容器运行时"
date: 2020-10-08T14:16:07+08:00
lastmod: 2020-10-08T15:09:29+08:00
draft: false
keywords: ['k8s容器运行时介绍']
description: "容器运行时笔记"
tags: ['kubernetes']
categories: ['containers']
autoCollapseToc: false
---

<!--more-->

## 1 标准

### 1.1 OCI

OCI（Open Container Initiative）标准是由 Docker 公司主导的一个关于容器格式和运行时的标准或规范，包含运行时标准（runtime-spec ）和容器镜像标准（image-spec）

* `runc`: OCI规范在Linux上的完整实现

### 1.2 CRI

k8s的容器运行时接口（CRI）。该接口是基于gRPC的，容器运行时只要实现了CRI，就能和k8s集成。

* `kubelet`的启动参数`--container-runtime`和`--container-runtime-endpoint`使用的容器运行接口
* `crictl`是CRI的客户端程序

## 2 docker

现在安装比较新的 docker（18.09.6），会看到实际上至少会有三个组件：runC、containerd、dockerd。

dockerd 是个守护进程，直接面向用户，用户使用的命令 docker 直接调用的后端就是 dockerd；dockerd 不会直接使用 runc，而是去调用 containerd；containerd 会 fork 出一个单独子进程的 containerd-shim，使用 runc 将目标容器跑起来。

Kubelet 则是通过内置的 docker-shim 去调用 dockerd。

```txt
+--------------------+
|                    |
|                    |  CRI gRPC
|   kubelet          +-----+                                                  +---------------+     +--------------+
|                    |     |                                                  |               |     |              |
|                    |     |   +---------------+       +--------------+ fork  |container-shim +----->  container   |
|      +-------------+     |   |               |       |              +------->               |     |              |
|      |             |     |   |               |       |              |       +---------------+     +--------------+
|      |            A+<----+   |               |       |              |                      runc(OCI)
|      | dockershim  |         |    dockerd    |       |  containerd  |       +---------------+     +--------------+
|      |             +--------->B              +------->C             |       |               |     |              |
|      |             |         |               |       |              +------->container-shim +----->  container   |
|      |             |         |               |       |              |       |               |     |              |
+------+-------------+         +---------------+       +--------------+       +---------------+     +--------------+
                                                                      |
                    A:unix:///var/run/dockershim.sock                 +------> ......

                                                        C:/run/containerd/containerd.sock
                                B:/var/run/docker.sock
```

## 3 cri-containerd

Containerd 也是 docker 公司实现的，后来捐献给了 CNCF。contianerd 把 dockerd 与 runc 解耦了，dockerd 不直接创建容器，而是通过 containerd 去调用 runc。从 contianerd 1.1 开始，contianerd 可以以插件的方式集成 CRI。contianerd 也可以使用除 runc 以外的容器引擎。

```txt
+--------------------+         +----------------------+          +---------------+     +--------------+
|                    |         |                      |          |               |     |              |
|                    |         |                      |  fork    |container-shim +----->  container   |
|   kubelet          |         |  containerd          +---------->               |     |              |
|                    |         |                      |          +---------------+     +--------------+
|                    |         |                      |                         runc(OCI)
|                    |         +--------------+       |          +---------------+     +--------------+
|                    |         |              |       |          |               |     |              |
|                    |         |  CRI-plugin  |       +---------->container-shim +----->  container   |
|                    |         |              |       |          |               |     |              |
|                    +--------->A             |       |          +---------------+     +--------------+
|                    |         |              |       |
|                    |         |              |       +---------->......
+--------------------+         +--------------+-------+

                                A:/run/containerd/containerd.sock
```

* `ctr`: containerd的客户端程序


## 4 CRI-O

CRI-O 是 RedHat 发布的容器运行时，旨在同时满足 CRI 标准和 OCI 标准。kubelet 通过 CRI 与 CRI-O 交互，CRI-O 通过 OCI 与 runC 交互，追求简单明了。可以看到，在这种方式下，就不需要使用 docker 了。

```txt
                                          +----------+          +--------------+
                                          |          |          |              |
                                          |  conmon  +---------->  container   |
+-------------+       +--------------+---->          |          |              |
|             |       |              |    +----------+          +--------------+
|             |       |              |                 runc(OCI)
|   kubelet   | CRI   |    CRI-O     |    +----------+          +--------------+
|             +------->A             |    |          |          |              |
|             |       |              +---->  conmon  +---------->  container   |
|             |       |              |    |          |          |              |
+-------------+       +--------------+    +----------+          +--------------+
                                     |
                                     +---->......

                       A:/var/run/crio/crio.sock
```

## 5 总结

* 容器运行时是管理容器和容器镜像的程序。有两个标准，一个是 CRI-runtime，抽象了 kubelet 如何启动和管理容器，一个是 OCI-runtime，抽象了怎么调用内核 API 来管理容器。标准实际上是定义了一系列接口，让上层应用与底层实现接耦。

* 实现 CRI 的 runtime 有 CRI-O、CRI-containred 等，CRI 的命令行客户端是 crictl。containerd 的客户端是 ctr。dockerd 的客户端是 docker。它们通过 unix sock 与对应的 daemon 交互。

* OCI 的默认实现是 runc。runc 是一个命令行工具，而不是一个 daemon。通过 runc 我们可以手动启动一个容器，也可以查看其他进程启动的容器