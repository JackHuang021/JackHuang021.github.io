---
title: Linux内核likely()和unlikely()
tags:
  - Linux
categories: Linux
abbrlink: 8c0c86d5
date: 2023-01-11 16:25:10
---

Linux内核中多次出现`likely()`，`unlikely()`的使用，他们的定义如下：
```c
#define likely(x)     __builtin_expect(!!(x), 1)
#define unlikely(x)   __builtin_expect(!!(x), 0)
```

<!-- more -->
`__builtin_expect`是GCC提供的内置函数（built-in functions），函数原型是
```c
long __builtin_expect(long exp, long c);
```
函数的返回值就是`exp`，不过他告诉编译器，代码期望的是`exp == c`， 如果`exp == c`条件成立的机会占绝大多数，那么程序运行性能将会得到提升，否则性能反而会下降

`builtin_expect()`用来引导gcc进行条件分支预测，在一条指令执行时，由于流水线的作用，CPU可以同时完成下一条指令的预取，这样可以提高CPU的利用率，在执行条件分支时，CPU也会预期下一条指令，但是如果预测的结果不为`exp == c`那么CPU预取的下一条指令就没用了，这样就降低了流水线的效率，跳转指令相对于顺序执行的指令会多消CPU时间，如果可以尽可能不执行跳转，也可以提高CPU性能。

表面上看`if(likely(value))`和`if(unlikely(value))`都等同于`if(value)`，也就是`likely()`和`unlikely()`作用是一样的，但是实际上执行的效果是不同的（从指令预测的角度上来说），加`likely`的意思是value为真的可能性更大一些，那么预测执行if的可能性要大些，`unlikely`正好相反。


加上这种修饰，编译成二进制代码时`likely`使得if后面的执行语句紧跟着前面的程序，`unlikely`使得else后面的语句紧跟着前面的程序，这样就会被cache预读取，增加程序的执行速度。