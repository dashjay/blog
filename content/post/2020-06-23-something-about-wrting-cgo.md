---
title: '写cgo(gobinding)的一些问题和解决方法'
date: '2020-06-23'
description: ''
author: 'dashjay'
---

## 综述

写 `cgo` 不是写出来的，而是调试出来的，想要给一个c库写 `gobinding` 往往会遇得到你意向不大到的困难。

- 内存泄露
- 类型转换
- 函数调用
- 非标准c接口（c++问题）

这里以我的经历开始记录一下，我给公司写一个动态库 `so` 的 `gobinding` 的过程。简述一下我曲折的道路：

- 一开始，公司只给一个 `so` ，和原来用 `python` 写的一个 `gobinding` ，我发现原来的py脚本用的类似于 `ffi` 的方法，就是动态 `load` ，总之我也用这种方法，但是无论如何不是这个类找不到就是哪个函数无法调用。看完教程之后我知道大家都有源文件，难度会容易一些。

- 问了前辈说能给提供一个.h头文件，遂开始写了。想

## 开始

### 在 `golang` 中引入头文件

如果头文件不是很长的话，建议直接写在一个文件里。

```golang
package goskynet

/*
#cgo LDFLAGS: -L${SRCDIR} -lskynet_external
#include ...
....
....
*/
import "C"
```

其中 `import "C"` 和其他的引入不能放在一个括号里，必须单独，并且必须贴紧上方注释。

### 将自定义的C类型 -> golang

目的是，在包外调用时无需再 **针对C相关的包** 申明任何东西，也就是无需再外部 `import "C"`

类型转化只要注意一个问题就好，就是类型是否经过 `typedef` 类型定义，如果经过了，在golang中无需指定struct

```c++
typedef struct skynet_t skynet_t; // c中typedef

type SkynetT C.skynet_t // golang 中
```

否则的话

```c++
struct skynet_t; // c 中未typedef

type SkynetT C.struct_skynet_t // golang 中
```

我在头文件中看到了，所有类都经过了typedef，无需我申明了。

于是我首先给所有类的用type定义到golang这边来。

### 翻译函数

把C的函数放到golang这边，并不难，但是在类型转化过程中会有一些困难。

#### 举例1

```c++
skynet_t *skynet_new(const skynet_conf_t *conf, const char *policy);
```

这个函数写到golang这边，首先返回值，两个参数都是指针

- 尽量每次操作指针都转换，golang并不会隐式的帮你转换类型
- 往c传时强制转换成c的类型，返回golang就相反
- 转化

  - 返回值强制转化(*SkynetT)()
  - 传入c强制转化`(*C.skynet_conf_t)(conf)`和`(*C.char)(unsafe.Pointer(&policy[0]))`

```golang
// SkynetNew wrap the func -> |skynet_t *skynet_new|
func SkynetNew(conf *SkynetConf, policy []byte) *SkynetT {
 return (*SkynetT)(C.skynet_new((*C.skynet_conf_t)(conf), (*C.char)(unsafe.Pointer(&policy[0]))))
}
```

#### 举例2

```c++
skynet_status_t skynet_classify(skynet_request_t *req,
skynet_classify_type_t type,
const void *data,
size_t length,
skynet_result_t *result);

```

这个函数有五个参数，分别是：
    - skynet_request_t(c类型)
    - uint32
    - void *
    - ulong(size_t)
    - skynet_result_t(c类型)

返回值是一个int32在调用它的过程中，搞出了很多问题（内存泄露、参数传输错误）

先写出结论来，再说到底怎么回事

```go
func (s *SkynetRequest) Classify(t uint32, data []byte, res *SkynetResult) int32 {
    payload := C.CBytes(data)
    defer C.free(payload)
    var status = C.skynet_classify(
        (*C.skynet_request_t)(s),
        (C.uint)(t),
        payload,
        C.ulong(len(data)),
        (*C.skynet_result_t)(res))
    return int32(status)
}
```

**第一个参数：** 首先，这个函数传了request作为第一个参数，我这应该是可以看做this指针，因此我将此方法，定义到该类型上，并且使用了指针：`func (s *SkynetRequest) Classify...`，这样可以方便使用s来作为函数的第一个参数

**第二个参数：** 是一个uint而已，但是为了安全仍然进行转换。

**第三个参数：** 是一个`const void *data`但是根据猜测，payload是一个字符串，**第四个参数：** 指定了他的长度，因此一开始我直接用了char也就是`C.CString()`传入，报错了`char as uchar`,但是却不知道`uchar`怎么传，网上各种骚办法，最后都没人说其实`uchar=byte`, 使用`C.CBytes(data)`传入完美解决，这里data的传入gc只会搞定`byte slice`，C那边的内存需要使用`free`来操作。

**第五个参数：** 是一个库内的指针，直接传入引用即可，但是一定要在传入前进行强制类型转换，否则也会报错。

其中没什么经验，很多都是一点一点试出来的，网络上资料很少，走了很多弯路，仍然有很多问题，这时候意识到自己基础的薄弱，有时候连个作用域都拿不准，不敢free又造成了内存泄露的问题出现，长路漫漫，加油。
