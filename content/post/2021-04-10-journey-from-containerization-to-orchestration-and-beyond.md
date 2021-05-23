---
title: "从容器化到服务编排"
date: 2021-04-10T10:36:25+08:00
date: '2021-04-10'
lastModified: '2021-04-10'
description: ''
author: 'dashjay'
tags: ["docker", "kubernetes"]
slug: journey-from-containerization-to-orchestration-and-beyond
---

声明：文章是非常主观的，并且作者完全是这方面的新手、

容器造就了更高级的服务端架构，和更复杂的部署技术。容器现在使用的如此的广泛，并且早已成为标准的规范([1](https://github.com/opencontainers/runtime-spec), [2](https://github.com/opencontainers/image-spec), [3](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/), [4](https://github.com/containernetworking/cni), ...)来描述容器不同的方向。当然，他建立在最低级的 Linux 原语之上，例如命名空间和 cgroup。但是容器化应用已经如此大规模的使用，如果不是他容器基础本身的分层，我们几乎很难实现它。我正在尝试使用持续的努力来引导我自己从最底层的开始直到最顶层，尽量多做练习（代码，安装，配置，整合等等），并且当然，尽可能有趣。本页的内容也会随着时间修改，映射着我对这个话题的理解。

### 容器运行时（Container Runtimes）

我想从最底层的非核心原语 —— **容器运行时**开始。**运行时**这个词在容器的知识中里有点模糊不清。每个项目、公司或者社区有他们自己的对容器运行时的理解。在大多数理解中，运行时这个关键词最小单元（创建命名空间，开始一个容器进程），有事被理解为非常复杂的容器管理，包含（但是不限于）容器操作，关于容器运行时的观点可以在[这个文章](https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r)里找到

![](/post/2021-04-10-journey-from-containerization-to-orchestration-and-beyond/runtime-levels.png)

这部分是专门用于底层的容器运行时，一大堆专家们组成了一个 [开放容器计划（Open Container Initiative）](https://www.opencontainers.org/) 标准化了底层的运行时，在 [OCI 运行时规范（OCI runtime specification）](https://github.com/opencontainers/runtime-spec)中。长话短说，一个底层运行时就是一个程序传入了一个文件夹作为容器的根文件系统，一个配置文件描述了容器的参数（例如资源限制，挂载点，启动哪个进程等等），并且结果就是一个运行时启动了一个工作负载。UPD：通常，一个工作负载就代表一个容器，也就是[一个独立的受限制的 Linux 进程（isolated and restricted Linux process）](https://iximiuz.com/en/posts/not-every-container-has-an-operating-system-inside/)。OCI 运行时的有点就是没有特别指定每一个部分的确切实现。而是定义了和他沟通的方式。[下面可以选阅，更详细的运行时实现](https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/#secure-container-runtimes)。

2019年，最广泛使用的运行时就是 [runc](https://github.com/opencontainers/runc)，这个项目开始于 Docker(因此用 golang 实现) 的一部分，但是最终被提取并且转化成了一个自治的 CLI 工具。不能低估这个组件的重要性—— runc 是 OCI 运行时的一个基本参考。在这个教程中，我们将会使用 runc 来完成一些工作，这是一篇[介绍文章](https://iximiuz.com/en/posts/implementing-container-runtime-shim/)。

![runc](/post/2021-04-10-journey-from-containerization-to-orchestration-and-beyond/runc.png)

另一个值得注意的 OCI 运行时的实现是 [crun](https://github.com/containers/crun)。它使用 C 实现并且被同时作为可执行文件和 c 库来使用。他起源于 Red Hat 和其他 Red Hat 项目，比如 buildah 或者 podman 使用 `crun` 而不是 `runc`。

UPDATE aug 2020：以前，Docker 提倡：“一个容器，一个进程”的模型，但是不断有一些需求的到来，让容器更加像传统的虚拟机（带有 `systemd` 或者像有一个管理员在里面），`runc` 官方从来没有对此提供支持（依然可以通过运行定制的高权限容器来实现）。幸运的是，有一个新的 OCI 兼容的容器运行时叫做  [sysbox](https://github.com/nestybox/sysbox) 希望解决这个问题。

> Sysbox 是一个开源的容器运行时（又叫 runc ），原来由 NestyBox 开发，可以开启 Docker 容器来作为虚拟服务，能够像 Systemd 那样服务，Docker 和 Kubernetes 在里面，容易使用，并且有不错的独立性。这使得你可以以新的方式来使用容器，并且提供了一个更快的，在许多场景中，更高效并且更可移植的虚拟机。

### 【更多】安全容器运行时

传统的容器是轻量的、快、并且方便的。但是他们有一个重要的问题。因为容器本质上就是一个独立并且受限制的 Linux 进程，只要有一个内核漏洞，就能让容器的代码访问宿主机。根据环境的不同，这可以成为一个很严重的问题。（例如用户在多租户集群上访问潜在的恶意代码）。

传荣的虚拟机提供了一个可选并且更加安全的（通过较高的独立性）方式来跑任意的代码。但是这样的代价很高的，每一个工作负载的开销都很大（CPU、RAM、磁盘空间），并且启动时间也非常慢（十多秒，相比容器通常都是毫秒级别）。

你可以了解这个困境，在[这篇文章中](https://unit42.paloaltonetworks.com/making-containers-more-isolated-an-overview-of-sandboxed-container-technologies/)。

幸运的是，有许多项目正在试图制造轻量的，像容器一样的虚拟机。不止一次要感谢标准化的力量，才有这些 OCI 兼容的实现。他们可以被用来作为[顶层运行时](https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/#container-management)

这个领域中值得注意的项目是：[Kata Container](https://katacontainers.io/)，[Firecracker](https://github.com/firecracker-microvm/firecracker)（通过 [Firecracker-Containerd](https://github.com/firecracker-microvm/firecracker-containerd)），和 [gVisor](https://github.com/google/gvisor)。

项目之间的实现细节非常不同，但是方法主要是放另一个 Linux-ish 层（一个优化过的 VM，或者 用户空间的内核实现）在 宿主机系统的顶层。每一个沙箱的，仿虚拟机环境可以有更多的工作负载在内部工作。因为实现是高度的优化，每一个工作负载非常小的，并且启动时间低于一秒。

- [Firecracker-containerd](https://lwn.net/Articles/775736/) 是一个 [firecracker microVM](https://github.com/firecracker-microvm/firecracker)(一个高度优化的 QEMU，使用 50kLOC rust)，运行在一个客户操作系统上，和一些 `runc-driven` 容器在内部，整合了一个自定义的 [containerd](https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/#containerd) 通过一个定制的 containerd-shim。
- [kata contaienrs projects](https://github.com/kata-containers/kata-containers) 是一个前途无量的视图将一个抽象优化的 hypervisor（KVM，QUME、或 [Firecracker](https://github.com/kata-containers/runtime/releases/tag/1.5.0)）运行在特定的操作系统上，作为一个虚拟机。每个虚拟机中都有一个 agent 来生产工作负载([架构概述](https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture.md))。他们是能与 OCI、[CRI(<https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/#cri-o>)] 和 [containerd](https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/#containerd) 兼容的([sourece](https://github.com/kata-containers/runtime#introduction))。
- [gVisor](https://github.com/google/gvisor) 运行在 golang 编写的用户空间应用内核当中，其实就是实现了一个必须使用的 Linux 系统调用的子集。他提供了一个 runsc（非常像 runc ） OCI 兼容的运行时。但是因为它完全运行在用户空间，它的运行时消耗非常高。

想知道这些项目的更多细节可以[查看这篇文章](https://www.inovex.de/blog/containers-docker-containerd-nabla-kata-firecracker/)

### 容器管理

使用 runc 在命令行中，我们可以启动非常多的容器，如果我们想的话。但是如果我们需要自动管理这些进程该怎么办呢？想象一下我们需要启动十多个容器保持跟踪他们的状态。他们中的一些需要在失败的时候被重启，资源需要再终止的时候被释放，镜像必须从 registries 拉回，跨容器的网络需要被配置，还有很多其他需求。这早就到达了一个更高的级别应用，他应该负责容器的管理。说实话，我不知道这个术语是否常用，但是我发现以这种方式构建事物是非常方便的。我会将下面的程序分类为容器管理者：[contianerd](#containerd)、[cri-o](#cri-o)、[dockerd](#dockerd) 和 [podman](#podman)。

#### containerd

As with runc, we again can see a Docker's heritage here - containerd used to be a part of the original Docker project. Nowadays though containerd is another self-sufficient piece of software. It calls itself a container runtime but obviously, it's not the same kind of runtime runc is. Not only the responsibilities of containerd and runc differ but also the organizational form does. While runc is a just a command-line tool, containerd is a long-living daemon. An instance of runc cannot outlive an underlying container process. Normally it starts its life on create invocation and then just execs the specified file from container's rootfs on start. On the other hand, containerd can outlive thousands of containers. It's rather a server listening for incoming requests to start, stop, or report the status of the containers. And under the hood containerd uses runc. (UPD: Actually, containerd is not limited by runc. See the UPDATE below). However, containerd is more than just a container lifecycle manager. It is also responsible for image management (pull & push images from a registry, store images locally, etc), cross-container networking management and some other functions.

和 `runc` 相同，我们能再次看到 Docker 的遗产 ———— containerd 曾经是 Docker 项目的一部分。如今，containerd 是另一个自给自足的软件。经管他自称为容器运行时，但是很明显，它和 runc 不是同类。containerd 和 runc 不仅仅是功能不同，并且组成方式都不同。runc 只是一个命令行工具，containerd 是一个守护进程。一个 runc 的实例不可能存活超过它底层的容器进程。通常它在创建的时候调用，并且开始时仅仅从容器的根文件系统执行特定文件。然而，containerd 可以比成百上千的容器活得更久。
