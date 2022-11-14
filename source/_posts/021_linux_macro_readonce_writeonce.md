---
title: Linux内核 READ_ONCE WIRTE_ONCE宏
tags:
  - Linux
  - Kernel
categories:
  - Linux
---

+ 在Linux内核代码中，经常可以看到读取一个变量时，不是直接读取的，而是需要借助一个叫做`READ_ONCE`的宏；同样，在写入一个变量的时候，也不是直接赋值的，而是需要借助一个叫做`WRITE_ONCE`的宏。这两个宏定义在``