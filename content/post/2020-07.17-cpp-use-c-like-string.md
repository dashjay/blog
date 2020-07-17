---
title: '使用c-like string strcat 的研究'
date: '2020-07-17'
description: ''
author: 'dashjay'
---

## 0x1楔子

几天前在学习cpp的时候，因为好奇，在上面试了一些 `c-like string` 拼接操作，一下击穿了我自己，我写了个样例。

```c++
#include<cstring>
#include<iostream>

int main(){
    char Hello[64]{"Hello World"};
    char temp[]{"Alex"};
    std::strcat(temp,Hello);
    std::cout <<temp[5]<<std::endl;
}
```

这段代码在gnu-gcc上直接编译不会过，但是我用的`mac llvm`，编译过了并且输出e。
这时候我就很奇怪了，`temp`应该只是一个 `['A', 'l', 'e', 'x', '\n']` 这样一个字符串，但是为什么strcat之后，却能”越界“访问？

为此，我提出了灵魂拷问：

- strcat底层是如何操作的？
- 这个不安全的操作被llvm优化了么？
- 明显的越界操作为什么得到了预期的结果？
- 如果第二个参数cat到第一个参数不够大，那么会重新分配，这对于cpp来说是否是一种内存回收操作？

问题有点天马行空，但是一个一个来看看吧。

## 0x2 strcat 底层如何操作

```c++
#ifndef STRCAT
# define STRCAT strcat
#endif
/* Append SRC on the end of DEST.  */
char *
STRCAT (char *dest, const char *src)
{
  strcpy (dest + strlen (dest), src);
  return dest;
}
libc_hidden_builtin_def (strcat)
```

立马看了gnu里定义的源代码。
并且，到知乎提问，不一会就有好几个大佬来回答。

大家的回答让我意识到，这不是cpp层的问题，而是一个内存，栈空间级别的问题。

## 0x3 这个不安全的操作被llvm优化了么

一句话打回来: 研究 未定义行为（UB）是毫无意义的

## 0x4 明显的越界操作为什么得到了预期的结果

```text
low address --------------------------- high address
             | | | | | | | | | | | |
            ---------------------------
             ↑           ↑
            temp        Hello
```

问题在于“恰好能完成这个操作”,这基于 temp (destination 参数) 后声明,它通常（或者）大概率被编译器分配到地址更低的位置。

## 0x5 cpp内存回收`?`

我是傻逼

## 0x6 总结

大佬们的建议普遍就是摈弃`cstr`那套的东西，`string` 的 `operator+` 就非常完美。

我表示赞同，但是这是在学习过程中，如果一旦到了生产中，难免接触祖传代码。
