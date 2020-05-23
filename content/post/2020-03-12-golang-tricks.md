---
title: golang_tricks
author: dashjay
date: '2020-03-12'
slug: golang-tricks
categories:
  - golang
tags:
  - go
---
### 1. 切片传参是引用？
```

func main() {
    ages := []int{6, 6, 6}
    fmt.Printf("原始slice的内存地址是%p\n", ages)
    modify(ages)
    fmt.Println(ages)
}

func modify(ages []int) {
    fmt.Printf("函数里接收到slice的内存地址是%p\n", ages)
    ages[0] = 1
}
//原始slice的内存地址是0xc0000d61c0
//函数里接收到slice的内存地址是0xc0000d61c0
//[1 6 6]
```

如果不能理解上面的这个，我再那个Int的例子来看看(我当时一直没有理解这个。。。)

```
func TestRef(t *testing.T) {
	i := 10
	fmt.Printf("i的存在的地址是%p\n", &i) // i的存在的地址是0xc0000a6080
	temp := &i
	fmt.Printf("temp的地址是%p\n", &temp) // temp的地址是0xc0000a0028
	Modify(&i)
	fmt.Printf("after Modify,i的值:%d\n", i) // after Modify,i的值:100
}

func Modify(i *int) {
	fmt.Printf("传入的地址是%p\n", &i)// 传入的地址是0xc0000a0030
	*i = 100
	//i = 100 // 为什么不让用这种方式，是因为i本身是一个指针，地址，存的地址指向了传入的那个值
}
```
所有指针都有自己的地址，但是他们都存着指向10那个地址。
![](/post/2020-03-12-golang-tricks_files/image-20200312223351464.png)


如果你会发现，golang里的使用make创建的map他也是一个指针，chan也是
```

func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
...
func makechan(t *chantype, size int64) *hchan {

```

最终我们可以确认的是Go语言中所有的传参都是值传递（传值），都是一个副本，一个拷贝。因为拷贝的内容有时候是非引用类型（int、string、struct等这些），这样就在函数中就无法修改原内容数据；有的是引用类型（指针、map、slice、chan等这些），这样就可以修改原内容数据。

诶，我一开始只是想看看为啥，特喵的切片传参那么奇怪

Go里只有传值（值传递）。你所看到的传引用只是特喵的把自己的指针传进去了。

### 2. Golang Test BDD

```
import (
	. "github.com/smartystreets/goconvey/convey"
	"testing"
)

func TestBDD(t *testing.T) {

	Convey("Given 2 even numbers", t, func() {
		a := 2
		b := 4
		Convey("When add the two numbers", func() {
			c := a + b
			Convey("The the result is still even", func() {
				So(c%2, ShouldEqual, 0)
			})
		})
	})
}
  Given 2 even numbers 
    When add the two numbers 
      The the result is still even ✔

```
这是个好东西

 
### 3.golang 中的chan 关闭之后，千万不要往里写东西，会panic

读端可以通过 `data, open <- channel`来知道chanel是否已经关闭
```
func Service() chan string {
	s := make(chan string)
	go func() {
		fmt.Println("start service")
		time.Sleep(time.Microsecond * 99)
		s <- "ok"
		fmt.Println("service exit")
	}()
	return s
}
func TestChannel(t *testing.T) {
	l := Service()
	select {
	case d := <-l:
		fmt.Println(d)

	case <-time.After(time.Microsecond * 100):
		t.Error("time out")
	}
}
```

### 4. const里也有好玩的，两种申明方式
```
const (
	Monday = iota + 1
	Tuesday
	Wednesday
)

// iota is a predeclared identifier representing the untyped integer ordinal
// number of the current const specification in a (usually parenthesized)
// const declaration. It is zero-indexed.
const iota = 0 // Untyped int.
```
这种会让整个const从1开始增长

估计有人已经想到下面这种骚操作了。
```
const (
	Readable = 1 << iota
	Writeable
	Executable
)
```
这个可以来判断权限之类的
```
    a := 0b001
    t.Log(a&Readable == Readable, a&Writeable == Writeable, a&Executable == Executable)
```
### 4. golang recover
这是一个不太常见的操作，有时候可以帮助服务出现异常的时候恢复正常，不过不建议常用
大佬都说要let it crash, 这种recover仅限于小规模使用
```
func TestPanicVsExit(t *testing.T) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("recovered from", err)
		}
	}()

	fmt.Println("start")
	panic(errors.New("something error"))
}
```

### 5. golang的装饰器
py里的装饰器有时候惹得人羡慕，，不过golang也可以做到，只不过看起有点难受

```
func timeSpent(inner func(op int) int) func(op int) int {
	return func(n int) int {
		start := time.Now()
		ret := inner(n)
		fmt.Println("time spent:", time.Since(start).Seconds())
		return ret
	}
}
```
golang里的函数，可以作为函数返回值，装饰器正是利用了这一点。上方的函数就是可以让一个函数显示运行事件，不过，必须是指定格式的。
```
func slowFun(op int) int {
	time.Sleep(time.Second * 1)
	return op
}

tsSF := timeSpent(slowFun) //tsSF 就是装饰后的函数

```

### 6. ping了之后会发生什么？
有同学问题ping ip之后会发生什么，我突然意识到，自己并不会。
按照常理分析应该是，查询路由，查看是否可达。
1. 先查看自己的arp表，如果命中了就发送ICMP请求
2. 如果没有命中（缺省）就会递送给网关，然后由网关来操作。
3. 不行的话，就不可达。

### 7. golang ConcurrentMap sharding的时候怎么看如何分配。

默认是32，可能
> 在上英语：有两个单词就记在这里了。
> 
> `feigns` 假装 比pretend号，后接名词 （feidns madness） 假装生气
> 
> `duel` 决斗
>
> `fortified` 强化
>
> `harrows`（耙子），使动用法（让人害怕）,it harrows me 
> 
> `fantasy` 幻想？

> `dumb bell`哑铃 `dump`哑
> 
> `relieve` 减轻
>
> `corw` v. 打鸣；自鸣得意；欢呼
>
> `impart` 传授

### 8. 
正在被执行的 goroutine 发生以下情况时让出当前 goroutine 的执行权，并调度后面的 goroutine 执行：

IO 操作
Channel 阻塞
system call
运行较长时间

### 9. 读写锁，当有写锁在等待读锁读完的时候，不能再增加读锁，否则会产生饥饿，所以下面这个代码死锁。
```
package main
import (
	"fmt"
	"sync"
	"time"
)
var mu sync.RWMutex
var count int
func main() {
	go A()
	time.Sleep(2 * time.Second)
	mu.Lock()
	defer mu.Unlock()
	count++
	fmt.Println(count)
}
func A() {
	mu.RLock()
	defer mu.RUnlock()
	B()
}
func B() {
	time.Sleep(5 * time.Second)
	C()
}
func C() {
	mu.RLock()
	defer mu.RUnlock()
}
```
A和C都想读，但是A和C读的中间，杀出一个写锁来，导致C不让读，A不能return，则defer也不能执行，然后就不了了之了。（死锁）

### 10. 一旦waitgroup执行到了wait之后，就不可以再执行add，会panic
```
package main
import (
	"sync"
	"time"
)
func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		time.Sleep(time.Millisecond)
		wg.Done()
		wg.Add(1) // <--panic
	}()
	wg.Wait()
}
```