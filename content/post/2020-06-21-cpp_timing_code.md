---
title: "c++ 测算函数运行时间"
date: '2020-06-22'
author: dashjay
---

### 如何测算函数运行时间

每次要测了，我都百度一串代码复制粘贴到我的代码中，时间长了我依然记不得，到底怎么测算的，大概是用clock_t啥之类的吧

这次我来找一个比较精准的测算方法。

### chrono库

这个库比较神秘，但是我上网查阅资料之后把它封装成一个类了。

```c++
#include <chrono> // for std::chrono functions

class Timer
{
private:
 // Type aliases to make accessing nested type easier
  using clock_t = std::chrono::high_resolution_clock;
  using second_t = std::chrono::duration<double, std::ratio<1> >;

 std::chrono::time_point<clock_t> m_beg;

public:
 Timer() : m_beg(clock_t::now())
 {
 }

 void reset()
 {
  m_beg = clock_t::now();
 }

 double elapsed() const
 {
  return std::chrono::duration_cast<second_t>(clock_t::now() - m_beg).count();
 }
};

```

之后通过如下方法来直接使用

```c++
int main()
{
    Timer t;
    // Code to time goes here
    std::cout << "Time elapsed: " << t.elapsed() << " seconds\n";
    return 0;
}
```
