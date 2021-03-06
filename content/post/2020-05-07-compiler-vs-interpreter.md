---
title: 编译器 VS 解释器
author: dashjay
date: '2020-05-07'
slug: 'compiler-vs-interpreter'
categories: []
tags: []
---




今天看了[一篇关于解释器和编译器的区别的文章](https://www.learncpp.com/cpp-tutorial/introduction-to-programming-languages/)，自己也谈谈自己的理解。

## 编译器

编译器是一个程序（可以使用其他语言来实现），可以读取 **源代码（source code）** 并且产生 **可执行文件**，之后仅需要运行可执行文件即可，无需在每次执行的时候编译（有一些程序编译器来是相当耗时的，笔者因为一些特殊原因编译了一次 `g++`，竟然花费了长达2小时）。以下图一就是编译器的运行过程。

<div align="center">
 <img src=/post/2020-05-07-vs_files/image-20200507132517333.png width="700px">
 <p> 图1：编译器运行过程</p>
</div>

流行使用编译器之前，编译器非常慢，并且从来不对产生的代码进行优化，许多年过后，编译器已经能非常快速的的优化并且生成二进制可执行文件，总之比人写汇编快一些。

## 解释器

解释器是一个直接从源代码中执行指令的程序，常见的解释语言有(Python，PHP)，解释型语言往往有更加灵活的语法相对于编译型语言，但是他们通常更加低效，因为每次执行的时候都需要解释。

<div align="center">
 <img src=/post/2020-05-07-vs_files/image-20200507133427000.png width="700px">
 <p> 图2：解释器运行过程</p>
</div>

## 编译器VS解释器

你可以幻想如果一方有压倒性的优势，另一方就没有存在的意义了，所以两者既然都存在，那么肯定有原因，

<https://stackoverflow.com/questions/38491212/difference-between-compiled-and-interpreted-languages/38491646#38491646>

翻译来自以上问答。

大体上讲，编译型语言或者编译器具有以下 **优点**：

1. 因为编译器可以在执行前浏览一遍代码，他们可以在生成最终可执行文件的之前，执行一些分析和优化，使得编译的可执行文件相比于逐行解释、执行来的更加快速。

2. 编译器可以生成与原来的高级语言表现相同的低级[^1]的code，……看不懂,大概意思就是可以减少内存的开销（我的垃圾英语）。

   > Compilers can often generate low-level code that performs the equivalent of a high-level ideas like "dynamic dispatch" or "inheritance" in terms of memory lookups inside of tables. This means that the resulting  programs need to remember less information about the original code,  lowering the memory usage of the generated program.

3. 编译产生的程序，运行起来比解释代码快，因为指令本身就是的目的，而不是像解析型语言一样，除了执行程序本身之外还要运行一个解释器。

大体上将，编译型语言或者编译器具有以下的 **缺点**：

1. 难以支持一些语言特性，例如动态语言类型，是很难去高效的编译。原因是编译器非不能预测会发生什么，直到程序运行（runtime），这意味可能不能产生较好的代码。

2. 编译型器往往需要一个较长的启动时间，因为对所有的代码执行分析花费较长的时间，这意味这像浏览器这样，需要快速执行快速得到结果的程序，往往会更慢，因为不一定会运行同一份代码多次。

   > Compilers generally have a long "start-up" time because of the cost of  doing all the analysis that they do. This means that in settings like  web browsers where it's important to load code fast, compilers might be  slower because they optimize short code that won't be run many times.

大体上讲，解释器或者解释型语言有以下几个 **优点** ：

1. 因为他们（解释型代码）可以在一边输入，执行，输出。没有必要去优化操作，他们企图有更加快速的启动方法。
2. 因为解释器知道程序在做什么在他们运行时，解释器可以执行一一些动态优化，这些都是编译器不能够看到的

大体上讲，解释器有以下的 **缺点**：

1. 解释器通常需要占用更多的内存，因为解释器需要保存更多程序的信息，运行时。

   > Interpreters typically have higher memory usage than compilers because  the interpreter needs to keep more information about the program  available at runtime.

2. 解释器通常需要花费一些CPU时间在程序运行时，会拖慢运行的程序。

## JVM

因为解释器和编译器有恰好互补的优缺点，因此，按照人类计算机发展的惯例[^2]，在两者中间的东西就会缓缓出现。`Java's JVM`就是这样的东西。JVM会去发现一些会执行多次的代码并且将他们翻译成机器语言，称为 ”hot code"，相反 ”cold code" 就直接解释运行。JVM 可以做一些动态分析，可以加速程序的运行。

## 其他

好像我写了一本书，美国人想要读有两种办法，他可以请一个翻译念给他听（解释器），也可以让另一个翻译把他翻译成英语（编译器）再从头看，当然不是像国内某些出版书籍一样，使用谷歌翻译。
