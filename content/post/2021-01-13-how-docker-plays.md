---
title: 'Docker 玩的障眼法'
date: '2021-01-13'
lastModified: '2021-01-13'
description: ''
author: 'dashjay'
tags: ["docker", "kubernetes", "Linux Namespace"]
slug: how-docker-plays
---

> 刚刚入手docker会感觉有点像虚拟机，但是又有许多不同，一查资料都在说虚拟机虚拟了磁盘/CPU...，docker轻量但是基本没说究竟怎么回事。
> 今天学习了 Docker 的一些实现细节之后，发现这些只不过都是 Docker 或者说 Linux 玩的障眼法，让我们来看一眼。

## 容器是什么？

容器其实就是在操作系统启动进程的时候，通过附加 **一些参数**，利用操作系统提供的接口来 **隔离不相关资源**，最后得到一个特殊的进程。

## Pid 障眼法

我们知道在 `Linux` 中，启动进程需要使用函数（系统调用），例如：`fork`, `vfork`, `clone` 等。下面给出一个 `clone` 的函数签名。

```c
int pid = clone(main_function, stack_size, SIGCHLD, NULL); 
```

使用这个函数，系统就会为我们创建一个新的进程，并且返回它的进程号 pid 。

```c
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```

如果我们在调用的时候，在参数中指定 `CLONE_NEWPID` 参数，在该进程内，就会看到一个全新的进程空间。

也就数说，这个进程会以为自己就是 1 号进程，但是其实这只是一个障眼法，在宿主机的进程空间里，进程的 PID 还是真实的数值。

这是 Linux 中的一个叫 Namespace的机制

## 障眼法的更优解释（命名空间）

所以当你理解了进程 Namespace 的时候，你就会明白，其实并没有容器。Docker 帮你启动你的进程的同时，使用了操作系统提供的命名空间限制，让你的进程以为自己在一个独立的空间中。

- Mount Namespace 让你的进程以为自己有根目录，其实那只不过是挂载了一个目录目录到进程的根目录。
- Network Namespace 让你的进程隔离于其他网路设备，只能看到你为它连接的网络设备。
- User Namespace 将本地的虚拟 user-id 映射到真实的 user-id 中。

包括就连主机名等等其他资源，也可以用障眼法隔离起来，只要理解了，就不会觉得 Docker 很神奇了。

## 弊端

1. 隔离不彻底，ubuntu14.04 有 docker 逃逸的 CVE...
2. docker 可能会占用非常多的资源，在后面要引入 Linux Cgroup 来限制容器使用资源。

## 参考资料

[Linux Namespace 简介 UTS](https://blog.lucode.net/linux/intro-Linux-namespace-1.html)

[极客时间-深入剖析Kubernetes-05-从进程说开](https://time.geekbang.org/column/article/14642)