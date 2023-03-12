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

到目前为止，我们也就是简单了解了一下，aws s3 协议请求和返回值，业务的话也相对简单：创建/删除 bucket，上传、删除 object。

然后就是遍历文件，对应的接口是 ListObjectV2，实现这个接口需要理解 object key 和 linux 树形文件的关系，说道这里要说一下这个 s3 和文件系统的区别。

s3 本质上是一个key-value存储，上传文件的业务就对应文件名为 `key = key_prefix/dir_name/file_name`，内容就是这个文件的content，**本质上 key 只不过是看起来符合 linux 树形文件系统** 的规范而已，如果你要把你的 key 当做是 linux 里的文件目录，key_prefix 是一个文件夹，`dir_name` 是一个文件夹，`file_name` 是一个文件的话，你也可以这么理解。s3 也可以帮你这样做，例如：

如果 S3 里面有这样一些文件：

```
key_prefix/dir_name1/file_name1
key_prefix/dir_name2/file_name2
key_prefix/dir_name3/file_name3
```

```
// 我下面的请求 `url` 里的一些字符没有经过 `urlEncode` 转化为 `html` 实体

GET /bucket_name?list-type=2&delimiter=/&prefix=key_prefix/ HTTP/1.1
x-amz-request-payer: RequestPayer
x-amz-expected-bucket-owner: ExpectedBucketOwner
```

如果上面上传的 key 存在，你就会得到 `dir_name1, dir_name2, dir_name3` 这样三个 `common_prefix`。

也就是说，当 `delimiter` 为 `/` 的时候，就会像文件系统那样，按照树形的去分割每个 key，像文件目录那样返回。

如果没有传入 delimiter，就直接返回所有key就好了……（当然一般会限制前1000个，如果传入 start-after 之后，会从第 start-after+1 个开始。

具体的实现在这里：[boss/pkg/s3backend/s3backend.go](https://github.com/dashjay/boss/blob/master/pkg/s3backend/s3backend.go#L211)


## 0x01

上面仅仅是对这个服务的一个简介而已，实际上的实现往往会考虑多个方面，从流程上来看，大概流程就是这样：

- 当用户发出 list 请求的时候，根据条件遍历所有的key，然后返回。
- 当用户发出 get 请求的时候，根据 key 查找对象，然后写回用户。
- 当用户发出 put 请求的时候，根据 key 把 body 写到对应的位置存储起来。

### 0b00 针对 list 的考虑

先说 list，如果用户的 bucket 下 key 足够多，索引如何简历，响应速度如何……

方案1：我们可以使用 mysql, sqlite, pgsql 等数据库服务，然后建一个表，把 key, content, bucket 等字段存进去，查询的时候也比较简单，如果是 prefix 查询，可以使用 like 语句。

方案2：如果是 key-value 的我们可以直接使用现成的库呢，例如纯golang 实现的 leveldb，基于其改造的 rocksdb, 还有etcd 用的 bolddb等，都是非常经典高性能的 key-value 库。场景也相当符合，一写多读，但是查找的时候如果需要遍历一遍才能找到对应的 prefix 的话，开销可能有点大。


### 0b01 针对用户发出的 get/put 的考虑

先说一下 put，我们可以定义一个良好的使用规范，但是我们不能提前对用户的行为有任何期望，以下是两个极端需要被考虑到的：

- 用户上传数百万个 key 很长，但是 content 很短甚至为 0byte 的文件。
- 用户上传一个 object 10T 的文件。

这两种情况处理起来都很复杂，第一种情况下，会极大的拖慢搜索遍历的速度，当 key 数量无限多的时候，几乎无法找到 key 对应的 内容大小。

当你在使用某种算法对key进行预处理，能够加快索引速度的时候，可能会有用户上传大量同类 prefix 的 key，例如： `collection-a-%s` 这样的同 prefix key 会造成你的算法失灵。

这时候可以借鉴，facebook 的 hatstack 等架构，对用户的小文件进行合并，具体可以参考论文：[Finding a needle in Haystack: Facebook’s photo storage](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf) 和知乎文章：[Facebook 如何存照片](https://zhuanlan.zhihu.com/p/99388774)

再说一下 get, 也存在一些极端：

- 某些热 key 和 冷 key 是否可以放到不同的介质当中，或者采用不同的压缩方式。如果一些存储归档了，很长时间不会读，可以采用压缩，或者换存储介质来节约成本。有些高频读到的 key，如果没有在索引角度优化的话，可能会成为集群横向扩张的瓶颈。


## 0x02

