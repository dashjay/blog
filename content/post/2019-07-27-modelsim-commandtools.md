---
title: 使用 Modelsim 中的 CommandTools 进行仿真
author: dashjay
date: '2019-07-27'
slug: modelsim-commandtools
categories: []
tags:
  - verilog
  - HDL
type: ''
subtitle: ''
image: ''
---

本文最后修改于 2020-12-31

> 本人在本科期间有学一门课程叫做数字逻辑，这是一门从与或门开始，一直到加法器等普通电子器件，最终实现一个 CPU 的课程。课程内使用的是一个叫 Vivado 的电路设计，仿真综合软件。我私下一直使用的是iverilog进行的仿真，生成vvp可执行文件之后运行得到结果，因为方便。

我一般会写这样的Makefile

```makefile
all: cmp vvp lxt

cmp:
 iverilog *.v -o tb_top.vvp
vvp:
 vvp tb_top.vvp

lxt:
 cat top.lxt

clean:
 @rm -rf tb_top.vvp tb_top.lxt
```

有了以上 **Makefile** ，将 ***.v**文件和 **Makefile** 放在一起，执行 `make` 命令即可完成相应操作

但是用这种方式，生成的波形只能使用GTKwave来查看，并且当有一些复杂的情况出现时候，iverilog就无法处理了，所以我们使用Intel提供的免费的Modelsim来完成仿真，仿真资料比较少，全是图形化界面的介绍，道路坎坷。

### 0x0：下载全套工具

我在 intel 官网下载了 lite 版本，最小套装，大同小异，包大小 100 多 M，这里就不留源文件了。
我下载的是：**Quartus-lite-18.1.0.625-linux.tar**

### 0x1：安装

解压之后有一键安装脚本，运行的时候会提示缺库，安装对应的库文件就可以使用了。

### 0x2：仿真出波形

#### 0x21. vlog工具

路径位于 **intelFPGA_lite/18.1/modelsim_ase/bin/vlog**
执行 **vlog *.v** 即可对当前目录所有 **.v** 文件进行编译操作

我给了一个模板 ***.v** 文件

```verilog
module top_module(
 input [3:0] a,
 input [3:0] b,
 output reg[7:0] p
);

 reg [7:0] pv;
 reg [7:0] ap;
 integer i;

 always@(*)
 begin
  pv = 8'b00000000;
  ap = {4'b0000,a};
  for(i = 0; i<=3; i=i+1)
   begin
    if(b[i]==1)
     pv = pv + ap;
     ap = {ap[6:0],1'b0};
   end
  p = pv;
 end
endmodule
```

运行**vlog *.v**命令即可在当前目录下生成一个work文件夹，里面有一些编译的成果，不过不用太在意文件的具体内容。

#### 0x22.使用vsim

命令是

```bash
vsim -c -do test.do top_module
```

这个test.do文件里面的内容有哪些呢？

```bash
> cat test.do
add list -hexadecimal /top_module/a
add list -hexadecimal /top_module/b
add list -hexadecimal /top_module/p

do sim.do

write list test.list

quit -f
```

不用太在意具体的内容，知道 **-hexadecimal** 是十六进制，去掉自动变成十进制。
下面有一个 **sim.do** 内容是

{{< highlight verilog >}}
add wave a
add wave b
add wave p
force a 16#0x8
force b 16#0xa
run 1000
force b 16#0x2
run 1000
force b 16#0x3
run 1000
force b 16#0x4
run 1000
force b 16#0x5
{{< /highlight >}}

相当于，将模块中的 **a,b,p** 接口抽离出来，接在sim文件当中，强行给了 **a,b** 输入，给模块提供了测试激励。

我写了一个Makefile 文件如下

```verilog
all: cmp sim cat

cmp:
 vlog *.v
sim:
 vsim -c -do test.do top_module
cat:
 cat test.list
```

那么我把我的文件目录压缩上传提供下载试试

![](/post/2019-07-27-modelsim-commandtools.en_files/1.jpg)

<div align="center">图1：示例代码运行结果</div>

[点击这里下载示例文件](/post/2019-07-27-modelsim-commandtools.en_files/finaltest.zip)

```bash
.
├── Makefile
├── sim.do
├── test.do
└── top_module.v
```

有了命令行工具也许你就可以只关心编程，而不同关心太多工具链相关的事情，这也是我研究这些乱七八糟的东西的目的。
