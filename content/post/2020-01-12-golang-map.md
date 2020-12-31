---
title: golang下map的性能分析
author: dashjay
date: '2020-01-12'
slug: golang-map
categories: []
tags: []
---

## golang 中 map 性能优化[低阶]

### 简单介绍

golang 中的 build-in 的 map 这个 map 是非线程安全的，但是也是最常用的一个家伙。
为了测试多个 map 的性能我写了个接口 Map

``` go
type Map interface {
 Set(key string, val interface{})
 Get(key string) (interface{}, bool)
 Del(key string)
}

```

然后这是封装的普通的 map

``` go
type OriginMap struct {
 m map[string]interface{}
}

func NewOriginMap() *OriginMap {
 return &OriginMap{m: make(map[string]interface{})}
}

func (o *OriginMap) Get(key string) (interface{}, bool) {
 v, ok := o.m[key]
 return v, ok
}
func (o *OriginMap) Set(key string, value interface{}) {
 o.m[key] = value
}

func (o *OriginMap) Del(key string) {
 delete(o.m, key)
}
```

别看一堆代码，其实就是 get 和 set 操作。在这里我们要使用 golang 自带的 test 工具

``` go
func TestOriginMaps(t *testing.T) {
 hm := NewOriginMap()
 wg := sync.WaitGroup{}
 for i := 0; i < Writer; i++ {
  wg.Add(1)
  go func(wg *sync.WaitGroup) {
   for k := 0; k < 100; k++ {
    hm.Set(strconv.Itoa(k), k*k)
    val, _ := hm.Get(strconv.Itoa(k))
    t.Logf("Get %d = %d", k, val)
   }
   wg.Done()
  }(&wg)
 }
 wg.Wait()
}
```

这其中有个变量 Writer 就是写者的数量，如果只有 1 的时候程序能安全运行退出

``` go
1264 ± : go test map_test/map_performance_test.go -v           ⏎ [3h1m] ✹ ✚ ✭
=== RUN   TestOriginMaps
--- PASS: TestOriginMaps (0.00s)
    map_performance_test.go:71: Get 0 = 0
    map_performance_test.go:71: Get 1 = 1
......
    map_performance_test.go:71: Get 99 = 9801
PASS
ok      command-line-arguments  0.339s
```

但是一旦我们把 Writer 数量改为 2

``` go
1264 ± : go test map_test/map_performance_test.go -v             [3h2m] ✹ ✚ ✭
=== RUN   TestOriginMaps
fatal error: concurrent map writes

goroutine 21 [running]:
```

立马就爆炸了。那么？？？？golang 自己官方心理没数么？

当然有数 golang 开发者其中之一可是拿图灵奖的。你可以点击[stackoverflow 上的讨论](https://stackoverflow.com/questions/11063473/map-with-concurrent-access "stackoverflow这里")和[github 这里](https://github.com/golang/go/issues/21035 "github 上的讨论")去查看相关的 issue

### Sync. Map

这是某大佬提出的解决方案，我们试试

``` go
type SyncMap struct {
 m sync.Map
}

func NewSyncMap() *SyncMap {
 return &SyncMap{}
}

func (o *SyncMap) Get(key string) (interface{}, bool) {
 v, ok := o.m.Load(key)
 return v, ok
}
func (o *SyncMap) Set(key string, value interface{}) {
 o.m.Store(key, value)
}

func (o *SyncMap) Del(key string) {
 o.m.Delete(key)
}
```

我简单封装了一下，测试个性能没啥问题。

现在把 Write 增加也没问题了，可是真的没问题么？

我们现在小改一下第一种 map 加了个 RW 锁，然后和这种 map 做一下比较看看？

``` go
type OriginWithRWLock struct {
 m map[string]interface{}
 l sync.RWMutex
}

func NewOriginWithRWLock() *OriginWithRWLock {
 return &OriginWithRWLock{
  m: make(map[string]interface{}),
  l: sync.RWMutex{},
 }
}

func (o *OriginWithRWLock) Get(key string) (interface{}, bool) {
 o.l.RLock()
 v, ok := o.m[key]
 o.l.RUnlock()
 return v, ok
}
func (o *OriginWithRWLock) Set(key string, value interface{}) {
 o.l.Lock()
 o.m[key] = value
 o.l.Unlock()
}

func (o *OriginWithRWLock) Del(key string) {
 o.l.Lock()
 delete(o.m, key)
 o.l.Unlock()
}
```

然后我们这次用 Test 里的 Benchmark 试试看，为了方便比较，我们写一个函数 benchmarkMap。

``` go
func benchmarkMap(b *testing.B, hm Map) {
 var wg sync.WaitGroup
 for i := 0; i < b.N; i++ {
  for j := 0; j < Writer; j++ {
   wg.Add(1)
   go func() {
    for k := 0; k < 100; k++ {
     hm.Set(strconv.Itoa(k), k*k)
     hm.Set(strconv.Itoa(k), k*k)
     hm.Del(strconv.Itoa(k))
    }
    wg.Done()
   }()
  }
  for j := 0; j < Reader; j++ {
   wg.Add(1)
   go func() {
    for k := 0; k < 100; k++ {
     hm.Get(strconv.Itoa(k))
    }
    wg.Done()
   }()
  }
 }
 wg.Wait()
}

func BenchmarkMaps(b *testing.B) {
 b.Logf("Writer: %d,Reader: %d", Writer, Reader)

 b.Run("SyncMap", func(b *testing.B) {
  hm := NewSyncMap()
  benchmarkMap(b, hm)
 })

 b.Run("map with RWLock", func(b *testing.B) {
  hm := NewOriginWithRWLock()
  benchmarkMap(b, hm)
 })
}

```

首先是 BenchMark 的函数当使用

``` go
go test .... -bench=. -benchmem
```

的时候会被调用，然后来测试两种 Map 性能，上面那个是测试性能的函数，分别对两个函数的进行测试~~拭目以待

> 当两者都是 100 的时候

``` go

 go test test_map/map_test.go  -v -bench=. -benchmem                                                                                                                         [14:12:59]
goos: darwin
goarch: amd64
BenchmarkMaps
    map_test.go:73: Writer: 100,Reader: 100
BenchmarkMaps/SyncMap
BenchmarkMaps/SyncMap-8                       80          13374265 ns/op         1710981 B/op      80867 allocs/op
BenchmarkMaps/map_with_RWLock
BenchmarkMaps/map_with_RWLock-8              100          12572631 ns/op          155019 B/op      16951 allocs/op
PASS
ok      command-line-arguments  3.323s

```

基本上 SyncMap 的整体性能是优于 mapWithRWLock 的我来分析一下为什么

从古至今，人们一直在时间和空间上做斗争，这次也不例外，两种锁的实现原理不一样。

<div align="center">
 <img src=/post/2020-01-12-golang-map_files/map-with-lock.jpg>
 <p> 图1：带锁的 map</p>
</div>

当我们使用普通 Map 带 RWMutex 会将整块内存锁住，然后其他请求就要等待。
SyncMap 是如何实现的呢？

<div align="center">
 <img src=/post/2020-01-12-golang-map_files/syncmap.jpg>
 <p> 图2：SyncMap</p>
</div>

它分为两块内存（存的都是指针），一块只读区域，一块 Dirty 区域支持读写。

两边的指针指向原数据，当需要 Get 的时候他会执行 **Load** 操作从 Read 中去获取指针指向的值，如果没有找到（ **miss** ）发生了，就转而会去 dirty 中获得数据并且存入 Read 中。

当 **miss** 超过一定数量的时候，他就会用原子操作把 dirty 的数据 Promote 到 ReadOnly 中。

因此 Sync 这种机制，往往只适用于 Key-Value 相对稳定的业务情况，读多写少的业务。

> 手痒想写个内存的看看到底多花多少内存
> go tool pprof 是一个工具可以查看代码测评产生的内存日志

``` go

go test map_test/map_performance_test.go -bench=. -memprofile=mem.prof
go tool pprof map.test mem.prof
(pprof)top
...
      flat  flat%   sum%        cum   cum%
    1.54GB 57.95% 57.95%     1.57GB 59.14%  command-line-arguments_test.benchmarkMap.func2
    0.43GB 16.22% 74.17%     0.69GB 25.94%  command-line-arguments_test.benchmarkMap.func1

```

不用说了这看起来三倍的内存消耗，果然越快内存越大。那么？本次测评到此结束？

！！！ 并没有！！！
还有一个大佬写了个 concurrent-map 甚叼，我们来观摩一波。[concurrent-map](https://github.com/orcaman/concurrent-map "concurrent-map")

立马封装一波

``` go
type ConCurrentMap struct {
 m cmap.ConcurrentMap
}

func NewConCurrentMap() *ConCurrentMap {
 conMap := cmap.New()
 return &ConCurrentMap{m: conMap}
}

func (c *ConCurrentMap) Get(key string) (interface{}, bool) {
 v, ok := c.m.Get(key)
 return v, ok
}
func (c *ConCurrentMap) Set(key string, value interface{}) {
 c.m.Set(key, value)
}

func (c *ConCurrentMap) Del(key string) {
 c.m.Remove(key)
}
```

迫不及待开始测试，当 Write=100，Reader=100 的时候

```bash
go test test_map/map_test.go  -v -bench=. -benchmem                                                                                                                         [14:24:48]
goos: darwin
goarch: amd64
BenchmarkMaps
    map_test.go:73: Writer: 1000,Reader: 1000
BenchmarkMaps/SyncMap
BenchmarkMaps/SyncMap-8                        8         129847762 ns/op        16410356 B/op     785167 allocs/op
BenchmarkMaps/map_with_RWLock
BenchmarkMaps/map_with_RWLock-8               10         117275854 ns/op         1723905 B/op     169971 allocs/op
BenchmarkMaps/CMap
BenchmarkMaps/CMap-8                          69          27681675 ns/op         1786702 B/op     169936 allocs/op
PASS
ok      command-line-arguments  4.424s

```

那么我同样做个表格吧，把读写的几种情况都列出来

| R/W | SyncMap | map_with_RWLock | CMap |
| --- | --- | --- | --- |
| 100/100 |`13657466` |`12122735` | `1946771`|
|10/100 |`4040595` |`11031799` |`1770506` |
| 100/10|`1696194` | `1916231`|`360180` |

最后说一下这个并发读map是怎么搞的

<div align="center">
 <img src=/post/2020-01-12-golang-map_files/cmap.jpg>
 <p> 图2：SyncMap</p>
</div>

左边是普通的map，当有读写的时候锁上了，其他线程就无法读写了。右边的是 **concurrentMap** ，他利用了一种 **partition** 的思想，把 **Map** 的内存 **SHARD** （分割）成N份，然后用不同的🔐锁上锁，那么降低了需要资源被锁的概率。

我们在日常中编程的时候容易陷入一种误区，就是这锁，那锁，全锁上，面试也在问各种锁，但是在真实QPS比价高的业务中，锁是一种很可怕的东西，如果能在编程的时候好好想想写出 **LockFree`的程序是最好的啊。

我是北京某211的混子，从19年10月开始写两行golang到现在不知不觉已经过去了2个月，上手就开始拉框架写代码的我已经进化到开始分析性能，然后优化代码啦，如果有小伙伴想一起讨论讨论，欢迎。
