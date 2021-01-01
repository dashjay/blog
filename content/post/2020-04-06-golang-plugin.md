---
title: golang新特性plugin模式
author: dashjay
date: '2020-04-06'
slug: golang-plugin
draft: true
categories:
  - golang
tags: []
---

苦于无法load一个c的so正在疯狂查资料的时候，突然发现？？1.14更新了一个新的模块--plugin

### 看起来十分黑魔法，应该是模仿，或者借助 [linux-dlopen](https://linux.die.net/man/3/dlopen)实现的

### 本文的例子在 <https://pkg.go.dev/plugin?tab=doc>

```go
// cat test.go
package main

import (
 "fmt"
)

var V int

func F() {
 fmt.Printf("Hello, number %d\n", V)
}

```

使用这个编译成.so文件 `go build -o plugin.so -buildmode=plugin test.go`

在另一个程序中使用使用

```go
package main

import (
 "plugin"
)

func main() {
 p, err := plugin.Open("plugin.so")
 if err != nil {
  panic(err)
 }
 v, err := p.Lookup("V")
 if err != nil {
  panic(err)
 }
 f, err := p.Lookup("F")
 if err != nil {
  panic(err)
 }
 *v.(*int) = 7

 f.(func())()
}

```

我本来以为这个库是用来调用c语言的，白高兴一场，我在cgo上被坑了，基本上无法使用。

现在高内存机器，已经不在乎编译出的可执行文件的尺寸了，但是大家还是偶尔会使用，这样可以在修改共享库的实现之后，不再重新编译主程序。

### 参考资料

1. [Go 语言设计与实现 8.1 插件系统](https://draveness.me/golang/docs/part4-advanced/ch08-metaprogramming/golang-plugin/)

1. Static Libraries vs. Dynamic Libraries <https://medium.com/@StueyGK/static-libraries-vs-dynamic-libraries-af78f0b5f1e4>
