---
title: "分享一下公司自维护的一个支持大 Object 的 lsm-tree kv 存储"
date: 2023-05-06T11:27:44+08:00
draft: false
slug: a-implemention-for-big-object-kv-base-on-lsm-tree
---

## 0x00 背景介绍

LSM tree 拥有无可匹敌的写入速度，主要原因是该模式将大量随即写入修改为顺序写入，相对于B族树，LSM更能进一步榨干HDD磁盘的写带宽，避免出现随机IO 到 HDD 上。根据文章 [深入浅出分析LSM树（日志结构合并树）](https://zhuanlan.zhihu.com/p/415799237) 中的介绍，LSM 并没有一种特定的实现方式，它的模式更像是这样：

> “磁盘顺序写” + “多个树(状数据结构)” + “冷热（新老）数据分级” + “定期归并” + “非原地更新”这几种特性统一在一起的思想

LSM tree 在结构上酷似 overlayfs 的 upper 和 lower 这样一种形式。但是具体实现起来各家都根据自己的需求来定制了五花八门的实现。最早的实现例如：RocksDB、LevelDB，以及后来者 Badger 等，有一些针对 Compaction，kv 分离，大 value 优化等。

上方文章给出了 LSM 树的定义：

> 1. LSM树是一个横跨内存和磁盘的，包含多颗"子树"的一个森林。
> 2. LSM树分为Level 0，Level 1，Level 2 ... Level n 多颗子树，其中只有Level 0在内存中，其余Level 1-n在磁盘中。
> 3. 内存中的Level 0子树一般采用排序树（红黑树/AVL树）、跳表或者TreeMap等这类有序的数据结构，方便后续顺序写磁盘。
> 4. 磁盘中的Level 1- n子树，本质是数据排好序后顺序写到磁盘上的文件，只是叫做树而已。
> 5. 每一层的子树都有一个阈值大小，达到阈值后会进行合并，合并结果写入下一层。
> 6. 只有内存中数据允许原地更新，磁盘上数据的变更只允许追加写，不做原地更新。

我仔细思考了一下，我对它的定义可以再稍微修改一下，不再描述为内存和磁盘，而是描述为：较快的介质和较慢的介质，所以我将整个 LSM Tree 的定义调整为如下：LSM tree 是一种横跨几种速度，容量不同的介质的数据索引和IO的pattern……，剩下的定义和上方的一样。

## 0x01 LSM tree 与分布式架构

最近转组到一个数据组，开始接触一些诸如分布式系统，Linux IO等相关的一些知识，学习组内一套基于 LSM tree 为核心的分布式存储架构的设计。大部分现有的 LSM tree 的实现通常都是单机的，接口大部分酷似 boltdb，都是单机的一个存储。


我们公司内部维护了一个几十 PB 的一个对象存储，是魔改了 LSM Tree 的一套系统，横跨 nvme，hdd 等不同速度容量的介质，个人认为设计的非常优雅，在此给大家介绍一下系统的设计。

## 0x02 L0

学习过 LSM tree 的同学知道，L0就是内存层，通常是使用跳表、红黑树这种数据结构，总的来说 L0 通常是容量较小，速度较快的介质。

通常庞大的对象存储系统，除了少量全闪集群之外，大部分系统都使用 HDD 作为存储介质，如果这时有 NVME 这种介质，那么相对来说他一定能作为 L0 存在。

所以这套系统的 L0 使用了一个磁盘上的全闪 NVME 存储，raft + boltdb 三副本实现索引的一致性，数据按照容错域，三副本放置在不同的三台机器上的 Object Store中，因为是全闪 ObjectStore + MultiRaft，可以支持极高的带宽的写入。

ObjectStore NVME 数据的写入几乎不会成为瓶颈。Raft 的 Apply 和 Commit 等后期可能成为瓶颈，需要合理的对数据进行分片，让索引的 IOPS 在可控范围内。https://github.com/dashjay/supreme-blob-storage，本人根据实现思想，进行了一个版本的简单实现，大概就是 raft（index）+ object store（fs base），实现非常简单。

## 0x03 L>1

学习过 LSM tree 的同学知道，L1层以上通常都是在磁盘上，存储的格式是一种类似 SST（sorted string table），是一种能够在磁盘上支持：get, list 操作的数据结构。

（to be continued）

