---
title: Golang也看看汇编
author: dashjay
date: '2020-03-23'
slug: golang-asm
categories:
  - golang
tags: []
---


```
// 源代码
package main

func main() {
	for i := 1; i < 10; i++ {
		println(i)
	}
}

```

```
// 汇编码
0x001d 00029 (main.go:3)    MOVL    $1, AX  // i := 1
0x0022 00034 (main.go:4)    JMP     78      // 调到78行
...
0x004e 00078 (main.go:4)    CMPQ    AX, $10 // 和10比较
0x0052 00082 (main.go:4)    JLT     36      // 条到36行
```

## 太多符号了根本看不懂

我们来把符号整理一下：

#### 符号

- FUNCDATA 和 PCDATA 做GC使用的，由编译器产生，暂时对我们无用

- PC，R0，SP是预定义对另一个寄存器的引用

- SB（static base）伪寄存器可以想象成一个内存地址，所以foo(SB)是一个由foo这个名字代表的内存的地址，当出现foo<>(SB)表示只有当前文件可见

- FP（frame pointer）伪寄存器是一个虚拟的栈指针，用来指向函数的参数。编译器维护了一个虚拟的栈指针，使用对伪寄存器offset的形式指向栈上的函数参数。传参的时候会出现，所以0(FP)，8(FP)就表示第一个和第二个寄存器（64位机器上）

- SP伪寄存器是一个虚拟的栈指针，用来指向**栈帧本地的变量**和**为函数调用准备参数**。他指向本地栈帧的顶部，所以一个队栈帧的引用必须是一个负数且范围在[-framesize:0]之间，例如：x-8(SP)，y-4(SP)，

#### 指令
- TEXT指令声明了符号runtime.profileloop。TEXT块的最后必须是某种形式的跳转，比如RET(伪)指令，参数是标志和栈帧的大小，是一个常量。
```
TEXT runtime·profileloop(SB),NOSPLIT,$8
    MOVQ    $runtime·profileloop1(SB), CX
    MOVQ    CX, 0(SP)
    CALL    runtime·externalthreadhandler(SB)
    RET
```
这个函数的栈帧大小为8字节（MOVQ CX 0(SP)操作栈指针），没有参数。

…………待续，我实在是看汇编到头痛。