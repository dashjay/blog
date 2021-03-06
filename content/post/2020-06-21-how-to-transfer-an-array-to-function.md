---
title: "c++ 中 如何向函数传入一个数组？"
date: '2020-06-21'
author: dashjay
---

### 如何向函数传入一个数组

这看起来是一个简单的问题，但是无论使用哪个方法，总是**不能同时兼顾**几个问题。

函数的参数在编译时，**参数**和**临时变量**等大小已经决定（栈空间大小），因此无法传入长度可能不定的一个参数。

```c++
void DoSth(int a[4]){
  //
}
```

> 即便你这样传入，你的编译器也会认为你传入的是一个`int *`。

```cpp
int arr[5] = {0,0,0,0,0};
DoSth(arr);
```

编译器也不会判你为错误，用一个词叫`退化(degrade)`,你的`int a[4]`会退化成为一个`int *a`

**两者究竟有什么区别？**

区别就是，没`退化(degrade)`之前，你可以知道它的长度，退化之后，便不得而知，通过以下case你可以检验我上面说的这句话

```cpp
int array[5]{ 1, 2, 3, 4, 5};
std::cout << sizeof(array) << "\n";  // 20 (在我的电脑上)
int *ptr{ array }; // 手动退化
std::cout << sizeof(ptr) << "\n"; // 8 (在我的电脑上)
```

> 为啥要说（在我的电脑上）：`int`在我的电脑上是4个字节，`int *`是8字节，也许在32位的电脑上会有所不同。

言归正传，回到问题本身，`C++为何不允许在函数中直接传递数组`：我大概明白你的意思，就是你想在不退化的情况下，直接传递一个数组包含的全部信息到函数中。

先说能不能做，再说为什么不推荐。

**可以:**

如果仅仅是`不退化`是可以做到的：

```c++
void Dosth1(int (&arr)[4]){}
```

这样就可以完成题目的愿望，或者你可以仍然认为我没有完成因为：

```c++
void Dosth1(int (&arr)[4]){
  std::cout<< typeid(arr).name()<<"\n"; // A4_i (在我的电脑上)
}
```

其实，他仍然是传送了一个指针到函数中，只不过保留了长度这个信息。因此我们可以在函数中使用range语句来实现迭代操作

```c++
void Dosth1(int (&arr)[4]){
  for(int &d : arr){
    std::cout << d << " ";
  }
}
```

**不推荐:**

我们本来可以传入`int a[5]`或者`int a[N]`到函数里，只需要多传一个长度参数即可，然而我们提前指定了长度，就失去了这个权利。

为什么c++想方设法的都要让你传入一个指针，而不是数组本身，那时因为栈空间非常昂贵，函数调用的时候要拷贝参数到寄存器，栈空间…………（说多了答主也不懂了）。
