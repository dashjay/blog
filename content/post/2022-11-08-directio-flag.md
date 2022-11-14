---
title: "DirectIO Flag 问题"
date: 2022-09-10T11:27:44+08:00
draft: false
slug: open-direct-io-flag
---

最近 Trans 到一个存储部门，老板希望我能改造一下块存储的IO模型，从 `blocksize=4M` 的 `sync` 写，到 `direct_io` 的写，看看性能有多少提升。我以前对这个 `flag` 的了解只有：它会跳过 `pagecache`，但是并不知道这个有多少作用，有哪些优点和缺点等等，今天查资料好好看一下，然后总结一下。

## 0x00 man open 

[open(2) linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)
里面有对 O_DIRECT 的描述如下：

```
Try to minimize cache effects of the I/O to and from this
file.  In general this will degrade performance, but it is
useful in special situations, such as when applications do
their own caching.  File I/O is done directly to/from
user-space buffers.  The O_DIRECT flag on its own makes an
effort to transfer data synchronously, but does not give
the guarantees of the O_SYNC flag that data and necessary
metadata are transferred.  To guarantee synchronous I/O,
O_SYNC must be used in addition to O_DIRECT.  See NOTES
below for further discussion.
```

大概翻译如下：

> 尽量减少 `I/O` 的缓存效应到或者从这个文件。通常这回降低性能，但是在特殊的场景下，例如当应用做他自己的缓存时。文件的 I/O 直接到/从用户空间的缓存。`O_DIRECT flag` 它本身尽力去同步的传输数据，但是并不保证像 `O_SYNC flag` 那样数据和元数据都传输完成。为了保证同步 `I/O`，`O_SYNC` 也必须在 `O_DIRECT` 的基础上添加。


理解：根据我们对文件系统的理解，平时我们的 write 操作把缓存里的东西拷贝到内核以后，需要等内核周期性的把脏页刷入磁盘，如果在这之前断电，我们的数据就丢失了。在打开文件的时候开启 `O_DIRECT` 之后，操作系统会跳过这个步骤，直接将文件放进磁盘中（尽量），按照字面描述的尽量，我理解这里面仍然有异步的可能性，例如：操作系统拿着缓存空间地址调用磁盘驱动等操作其实是异步的……（全凭臆想，没有看过操作系统相关的代码）

当 `O_SYNC` 开启之后，才会保证数据和元数据同时到达磁盘。


## 0x01 align

根据我们对文件系统的了解，pagecache 就是一些用来给磁盘做缓存的空间，如果操作系统要把某些 page 写入磁盘时，这些 page 都是 4k 对齐的。当我们希望跳过 pagecache，直接和磁盘做 IO 的时候，内存 4k 对齐就需要我们自己去完成了。

[Quora 提问：Why does O_DIRECT require I/O to be 512-byte aligned?](https://www.quora.com/Why-does-O_DIRECT-require-I-O-to-be-512-byte-aligned)

中 Robert Love 回答道：

> O_DIRECT requires that I/O occur in multiples of 512 bytes and from memory aligned on a 512-byte boundary because O_DIRECT performs .... usually larger than the sector size and 4KB. You can obtain a filesystem's logical block size with the BLKBSZGET ioctl.) 
> 
> --- Robert Love

我简单翻译一下：

> `O_DIRECT` 要求 `I/O` 发生于 512Bytes 的倍数，并且从512Bytes对其的内存块上，因为 O_DIRECT 进行了 Direct Memory Access(DMA) 直连了后端的存储，绕过了任何的缓存。进行不对其的传输会要求“清理” I/O，通过显式的对齐的用户空间的缓存，并且填补空隙（在 buffer 结束到下一个 512 bytes 倍数的空间），这样会消除使用 O_DIRECT 的优点。

> 从另一个角度来看这个问题，`O_DIRECT` 的美在于他砍掉了 `I/O` 过程使用的虚拟内存。没有从用户的内核空间拷贝，没有页缓存，没有 pages period。但是这意味着内核为你做所有小的事情——对齐是最大的一件你需要做的事情。底层的的存储期望所有的事情的都是以扇区（sector）为单位的，因此你也需要以扇区为单位和系统交流。

> 那也是为什么这个魔数不一定总是 512 Bytes。他是底层块存储的扇区的尺寸。这个数字大多数时候都是 512 Bytes 在标准的硬件盘上，但是也会变化。例如，ISO 9660，CD-ROMs 是一个标准的格式，支持 2048 bytes 的扇区大小。你可以使用 BLKSSZGET ioctl 操作来获取底层的存储的扇区大小。可移植的代码应该使用 ioctl 返回的值，而不是硬编码 512Bytes。

> 由于历史原因，在 linux 内核中把 sector size 作为魔数。之前，O_DIRECT的调用需要对齐操作系统的逻辑 block size，通常会币 sector size 要大。你可以通过 ioctl BLKSBSZGET 获取文件系统的逻辑块大小

## 0x02 在 Golang 中的使用

先不说 golang，单纯的 O_DIRECT 的使用需要在 Open 文件的时候将对应的参数传入，然后用户在此模式下的写入需要手动保证几条对齐规则：

- IO 的大小必须对齐扇区大小（512字节）
- IO 偏移（指针位置）必须按照扇区大小对齐
- 内存 buffer 的地址也必须是扇区对齐的；

手动实现以上条件比较麻烦，有开源的库：[https://github.com/ncw/directio](https://github.com/ncw/directio)，这个仓库通过一些 hack 操作，在 golang 里面实现了满足上方条件的方法。

### O_DIRECT 打开

```
rfp, err := directio.OpenFile(file, os.O_RDONLY, 0666)

wfp, err := directio.OpenFile(file, os.O_WRONLY, 0666)
```

### 读取数据
```
buffer := directio.AlignedBlock(directio.BlockSize)
_, err := io.ReadFull(rfp, buffer)
```

### 写入数据
wfp.Write(buffer)


## 参考

- [(Quora)Why does O_DIRECT require I/O to be 512-byte aligned?](https://www.quora.com/Why-does-O_DIRECT-require-I-O-to-be-512-byte-aligned)
- [(man page) linux man page](https://man7.org/linux/man-pages/man2/open.2.html)
- [知乎 | Go 存储基础 | 怎么使用 direct io ？](https://zhuanlan.zhihu.com/p/459311470)