---
title: 034-linux-kernel-api
date: 2023-01-18 14:22:33
tags:
  - Linux
  - Kernel API
categories: Linux
---

#### likely() unlikely()
Linux内核中多次出现`likely()`，`unlikely()`的使用，他们的定义如下：
```c
// include/linux/compiler.h
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

#### IS_ERR(), PTR_ERR()
首先来看定义
```c
// include/linux/err.h
// 使用一个4k的空间保存错误码
#define MAX_ERRNO   4095

// unlikely()在这里的用途，表示x大概率是一个正常的指针
// -4095转换为unsigned long时,值为0xFFFFF001(32位)或0xFFFFFFFFFFFFF001(64位)
#define IS_ERR_VALUE(x) \
unlikely((unsigned long)(void *)(x) >= (unsigend long )-MAX_ERRNO)

// 由错误码求指针，-1 -> 0xFFFFFFFF
static inline void * __must_check ERR_PTR(long error)
{
    return (void *) error;
}

// 由指针求错误码，0xFFFFFFFF -> -1，依次类推
static inline long __must_check PTR_ERR(__force const void *ptr)
{
    return (long) ptr;
}

// 判断x是不是一个错误指针
static inline long __must_check IS_ERR(__force const void *ptr)
{
    return IS_ERR_VALUE((unsigned long)ptr);
}
```

内核空间最后一个page是专门为错误码保留的，即内核用最后一页捕捉错误，一般不可能用到内核空间最后一个page的指针，因此在写设备驱动程序的过程中，涉及到的指针，必然有以下三种情况：有效指针、空指针、错误指针

所谓的错误指针就是指其已经到达了最后一个page，比如对于32位系统，内核空间最高地址为0xFFFFFFFF，那么最后一个page就是0XFFFF0000~0xFFFFFFFF（假设4k一个page），这段地址是被保留的，`IS_ERR()`就是用来判断指针是否有错，如果指针不是指向最后一个page那么没问题，如果指针指向了最后一个page，那么说明实际上这不是一个有效的指针，这个指针里保存的实际上是一种错误代码，很常用的方法就是先用`IS_ERR()`来判断是否是错误指针，再调用`PTR_ERR()`来返回这个错误代码，返回错误码的时候一般加一个负号

再来看看`__must_check`宏
```c
#define __must_check        __attribute__((__warn_unused_result__))
```
`warn_unused_result`如果具有此属性的函数的调用者不使用该函数的返回值，则该属性会导致发出警告
