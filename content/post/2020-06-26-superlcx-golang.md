---
title: 'golang实现超lcx功能'
date: '2020-06-26'
description: ''
author: 'dashjay'
---



> 今年有一个端口流量转发的需求
>
> 发请求者->转发器 ->
>
> - -> 业务1
> - -> 业务2

并且不仅是转发，需要他们在转发的同时就dump出一份来，否则用nginx就挺好（我知道可以给nginx写插件，但是我太菜了c基础不够好）

看了著名工具lcx之后，上手自己写一个，发现用`golang`的时候事情并没有那么复杂，只需要很简单的一些操作就能可以完成。

### copy模式

```go
conn, err := lis.Accept()
 if err != nil {
  log.Printf("[x] accept error, detail: [%s]", conn)
  return
 }
 conn2, err := net.Dial("tcp", target)
 if err != nil {
  log.Printf("[x] connect to target error, detail: [%s]", err)
  return
 }
 log.Printf("[+] transfer between [%s] <-> [%s]",
             conn.LocalAddr(), conn2.RemoteAddr())
 go func() {
  var wg sync.WaitGroup
  wg.Add(2)
  go func() {
   io.Copy(conn2, conn)
   wg.Done()
  }()

  go func() {
   io.Copy(conn, conn2)
   wg.Done()
  }()
  wg.Wait()
  conn.Close()
  conn2.Close()
 }()
```

核心仅仅这一段代码，即可完成，`goroutine`这个机制真的给我们提供了太多的便利。

### proxy模式

```go
 p := &httputil.ReverseProxy{
  Director:       s.director,
  Transport:      &myTripper{RoundTripper: http.DefaultTransport},
  ModifyResponse: s.modifyResponse,
 }
 panic(http.Serve(s.lis, p))
```

本模式大量基于http包来编写，主要工作在中间件开发上。

### blend模式

混合两种模式，并且创造出一个兼具两者行为优点的蜜罐。

以上的代码开源在[superlcx](https://github.com/dashjay/superlcx)
