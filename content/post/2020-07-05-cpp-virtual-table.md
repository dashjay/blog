---
title: 'cpp 的虚表到底是什么意思？'
date: '2020-07-05'
description: ''
author: 'dashjay'
---

## 0x1 Binding

想弄明白虚表是什么首先要清楚绑定(binding)的概念。

**绑定：** 可以通过标识符(identifiers)->地址(address)。标识符是函数和变量都有的，但是今天我们为了学习函数，因此只讨论和函数有关的内容。

有两种绑定，`early binding` and `late biding`.

## 0x2 Early Binding

编译时遇到要调用函数时，已经具体调用哪个函数，当遇到调用的某个函数的时候，编译器会用一条指令去替换这个函数，帮助CPU跳到地址所在的函数入口，开始执行，这种 `Binding`叫`Early Binding`，也就是编译的时候就确定的绑定模式。

例如著名的A+B问题，就是一个常见的Early Binding。

```c++
int add(int x, int y)
{
    return x + y;
}

int main(){
    add(3, 5); // 编译时绑定
}
```

## 0x3 Late Binding (Dynamic binding)

有时候，不到运行时（runtime）是不知道要调用哪个函数的，这就是所谓的 late binding, 我更喜欢它另一个名字 —— dynamic binding。s

那么在什么情况下，会出现 **后绑定** 的情况呢？

你想象一下，函数指针就是这样一种情况，不到运行时，永远不知道函数的具体是哪一个。

```c++
#include <iostream>
int add(int x, int y)
{
    return x + y;
}

int main()
{
    int (*pFcn)(int, int) = add;
    std::cout << pFcn(5, 3) << std::endl; // add 5 + 3

    return 0;
}
```

显而易见，这相对于 early binding 是更加低效的，但是这又可以将函数作为参数传递。

## 0x4 Virtual Table

为了实现虚函数，C++ 使用了特定的形式的一种 late binding，设计了一个虚表（Virtual Table）来实现它。

虚表只是一个查询表（LookUp Table），有很多名字，“vtable”, "virtual function table", "virtual method table" or "dispatch table"都是它的名字，我最喜欢最后一个名字，调度表 (dispatch table)。

每个类使用了virtual关键词的，都会得到一个虚表。这个表就是一个静态的函数表，每一个函数，都在虚表中有一个入口，指向最末尾（most-derived【继承链末端的】）函数。

### 0x41 *__vptr

每个类都会有有一个指针 `*__vptr` 指向虚表（类创建时自动创建）。

这个指针会指向虚表，并且表中存储这每个函数的入口。

```c++
class Base
{
public:
    virtual void function1() {};
    virtual void function2() {};
};
class D1: public Base
{
public:
    virtual void function1() {};
};
class D2: public Base
{
public:
    virtual void function2() {};
};
```

#### 1. Base 的 `*__vptr`

Base的`*__vptr` 指向了Base的虚表，上面有两个函数入口，`function1()` 和 `function2()` 分别指向了自己实现的两个virtual函数。

#### 2. D1 的 `*__vptr`

D1 同样包含一个 `*__vptr` 指向了 D1 的虚表，其中因为 D1 实现了 `function1()` ，所以虚表中 `function1()`  指向了自己实现的那个，而 `function2()` 指向了Base中的那个

#### 3. D2 的 `*__vptr`

D2实现了 `function2()` ，所以情况和D1相反。

画一张图就是这样

<div align="center">
 <img src=/post/2020-07-05-cpp-virtual-table/VTable.gif>
	<p>	图片1：带锁的map</p>
</div>


### 0x2 基类指针

```c++
int main()
{
    D1 d1;
    Base *dPtr = &d1;

    return 0;
}
```

虽然dPtr是一个Base指针，但是其 `__vptr` 依然指向了 D1 的 `__vptr` 。

那么当调用 `dPtr->functon1()` 的时候会发生什么呢？

- 发现 `function1()` 是一个虚函数
- 使用 `dPtr->__vptr` 来获得 D1 的虚表。
- 决定要调用哪一个 `function1()`

## 0x5 总结

通过这些表，运行时可以知道究竟要调用哪个函数，但是这个操作会花去一些时间，主要体现在以下两方面。

- 通过 `__vptr` 查找虚表
- 通过虚表定位要执行的函数
- 再执行

对比直接调用（一次操作），和指针间接调用（两次操作），虚函数的调用需要三次操作，因此是比另外两种情况稍微慢一些。

建议，虚函数很酷，很有用，但是他不但会使得你的类比之前的稍微大一点，还会消耗稍微更多的性能。
