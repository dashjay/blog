---
title: '为什么服务器突然变得非常慢无法做任何操作（学习）'
date: '2020-11-15'
description: ''
author: 'dashjay'
---

## 进程状态

在你输入 top 命令的时候，会看到很多信息，你知道这些信息的意思么？

```cpp
top - 14:59:02 up 58 min,  1 user,  load average: 0.00, 0.00, 0.00
Tasks: 103 total,   1 running, 102 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.2 sy,  0.0 ni, 99.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1989.6 total,    814.9 free,    398.8 used,    776.0 buff/cache
MiB Swap:   1022.0 total,   1022.0 free,      0.0 used.   1414.5 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1131 sddm      20   0  486264  77316  38168 S   0.7   3.8   0:22.06 sddm-greeter
 1492 root      20   0 1000288  88576  43648 S   0.3   4.3   0:10.11 dockerd
 1498 root      20   0  878428  33880  22912 S   0.3   1.7   0:09.85 docker-containe
    1 root      20   0  103992  10176   7796 S   0.0   0.5   0:01.41 systemd
    2 root      20   0       0      0      0 S   0.0   0.0   0:00.00 kthreadd
    3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
    4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par_gp
    6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/0:0H-k
```

今天我们关注 S 列：

他有几个不同的状态，R、D、Z、S、I 等几个状态，挨个看看：

- R 是 Running 或 Runable 的缩写，表示进程正在运行，或者在 CPU 的就绪队列中等待运行。

- D 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uniterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。

- Z 是 Zombie 的缩写，它表示僵尸进程，应该知道它的意思。它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没回收它的资源。

- S 是 Interruptible Sleep 的缩写，也就是可中断睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。

- I 是 IDLE 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上，I状态的进程不会导致平均负载升高。

还有个不常见的状态：T，这是你使用 Ctrl+Z 或者使用 Kill 向进程发送一个 SIGSTOP 信号，进程会暂停，你可以输入fg或者发送一个 CIGCONT 的信号给它，便能继续恢复执行。

X状态表示已经死亡，你不会在 top 里看到他。

## Case1

系统缓慢，大量僵尸进程，来不及回收会导致 pid 用完从而无法创建新的进程。

## Case2

大量进程处于 D 状态（不可中断），可能正在和硬件交互，也会导致缓慢。

## pidstat -d 查看磁盘情况
