---
title: "实现一个简单的对象存储"
date: 2022-09-10T11:27:44+08:00
draft: false
slug: implement-an-oss
---

最近 Trans 到一个存储部门，人比原来少，但是人狠话不多，最近学到的东西也不少，单独拿一个 OSS 出来分享一下。

我是下班时间跟着 [十三天精通超大规模分布式存储系统架构设计——浅谈B站对象存储(BOSS)实现](https://mp.weixin.qq.com/s/IJhXKZSa5SBiBgXg0dJkHg) 实操了一把，大概花了两周时间完成了上面五天的任务，是不是我太菜了，我擦……

具体代码在这里[https://github.com/dashjay/boss](https://github.com/dashjay/boss)

## 0x00 

Amazon Simple Storage Service (S3) 叫做对象存储服务。S3 是 Amazon Web Services (AWS) 提供的一项服务， 它通过基于 RESTful API 的接口提供对象存储。根据亚马逊的报告，到 2021 年，有超过 100 万亿个对象存储在 S3 中。

我还算比较熟悉 OSS，如果不熟悉的话需要花上半天时间研究一下这玩意儿。我简单介绍一下，有个工具叫做 awscli 里面有个 s3 子命令可以完成这样的操作：

```
aws s3 cp localfile.txt s3://dashjay/remotefile.txt
aws s3 copy localdir s3://dashjay/remotedir
```

他会把本地的文件通过某种方式传送到服务器上存起来，如果是非存储行业的人可能会感觉：我 ca，这不就是一个支持上传的 http server 么。

这就需要给介绍一下 s3 提供非常高稳定性，高可用，尺寸无限制的一个文档存储器。它并不能是一个支持上传的 http server 后面挂块磁盘就能解决的问题。

在云原生（云计算）概念里，没有所谓的硬盘啥的，你无法感知到后端的数据是如何存储的，同时数据的量通常是很大的（30PB）这样的数量级，一块普通的机械硬盘 9T 的话，需要 3414 块才能装下那么多数据，真么多块盘，怎么管理他们，盘坏了数据怎么办？

服务等级协议 (SLA)，SLA 是服务提供商和客户之间的协议。 AWS S3 对象存储，提供了 99.9 的可用性，以及夸张的 99.999999999% （11个9） 的数据持久性。

aws 的 s3 其实只是名字简单而已，协议似乎就是一个 key-value 存储，只不过 value 可能是一个比较大的文件，需要支持快速查找，但是它的实现肯定并不简单，我们的任务就是实现一个简单的版本。

首先需要学习协议相关的内容，amazon s3 的协议接口还是比较多的，我们只会实现其中一部分（完成一个较小的 MVP）

看一下下方的文档：

[API_Operations_Amazon_Simple_Storage_Service](https://docs.aws.amazon.com/AmazonS3/latest/API/API_Operations_Amazon_Simple_Storage_Service.html)

我们主要需要实现如下几个接口：
- [CreateBucket](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CreateBucket.html)
- [DeleteBucket](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteBucket.html)
- [ListBuckets](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListBuckets.html)
- [PutObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html)
- [ListObjectV2](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListBuckets.html)
- [DeleteObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html)
- [HeadOBject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html)

可以现观察一下这些接口的入参和返回值，我定义了一个 interface，把这些接口概括了一下，代码如下：[boss/pkg/interfaces/oss.go](https://github.com/dashjay/boss/blob/master/pkg/interfaces/oss.go)

我们需要根据协议的内容写出一个解析函数，可以帮助我们把 aws s3 的请求解析出来，这个过程繁琐，而且也不是关键，可以直接抄我的代码。

[boss/pkg/parse/s3query.go](https://github.com/dashjay/boss/blob/master/pkg/parse/s3query.go#L99)

```golang
func S3Query(r *http.Request) (q types.S3Query) {
	bucket, object := path2BucketAndObject(r.URL.Path)
	query := r.URL.Query()

	q.ListQuery.Version = 1
	q.ListQuery.Delimiter, q.ListQuery.Prefix, q.ListQuery.Marker, q.ListQuery.KeyMarker, q.ListQuery.VersionIDMarker =
		query.Get(Delimiter), query.Get(Prefix), query.Get(Marker), query.Get(KeyMarker), query.Get(VersionIDMarker)
    ....
}
```

## 0x01


## 0x02

