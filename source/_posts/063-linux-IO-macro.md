---
title: Linux内核中_IO, _IOR, _IOW, _IOWR宏解析
date: 2024-01-04 10:14:14
tags:
  - Linux
  - _IO
categories: Linux
---

在驱动程序里，ioctl()函数上传送的变量cmd是应用程序用于区别设备驱动程序请求处理内容的值，cmd除了可区别数字外，还包含有助于处理的几种信息，cmd的大小为32位，这里将这32位数据共分为4个域，其定义在`include/uapi/asm-generic/ioctl.h`中

Linux内核版本v5.10.153

这4个域分别为：
1. bit7:0 _IOC_NRBITS，共8位，用来表示命令的顺序序号
2. bit15:8 _IOC_TYPEBITS，共8位，魔数区，这个值用来区分和其它设备驱动程序的ioctl命令
3. bit29:14 _IOC_SIZEBITS，共14位，数据大小区，表示ioctl()中arg变量传送的内存大小
4. bit31:30 _IOC_DIRBITS，共2位，区别读写区，作用是区分是读取命令还是写入命令

```c
// include/uapi/asm-generic/ioctl.h
#define _IOC_NRBITS	8
#define _IOC_TYPEBITS	8

#ifndef _IOC_SIZEBITS
# define _IOC_SIZEBITS	14
#endif

#ifndef _IOC_DIRBITS
# define _IOC_DIRBITS	2
#endif

#define _IOC_NRMASK	((1 << _IOC_NRBITS)-1)          // 0x7f
#define _IOC_TYPEMASK	((1 << _IOC_TYPEBITS)-1)    // 0x7f
#define _IOC_SIZEMASK	((1 << _IOC_SIZEBITS)-1)    // 0x3fff
#define _IOC_DIRMASK	((1 << _IOC_DIRBITS)-1)     // 0x03

#define _IOC_NRSHIFT	0
#define _IOC_TYPESHIFT	(_IOC_NRSHIFT+_IOC_NRBITS)      // 8
#define _IOC_SIZESHIFT	(_IOC_TYPESHIFT+_IOC_TYPEBITS)  // 16
#define _IOC_DIRSHIFT	(_IOC_SIZESHIFT+_IOC_SIZEBITS)  // 30
```

区别读写区中的值定义如下：
```c
/*
 * Direction bits, which any architecture can choose to override
 * before including this file.
 *
 * NOTE: _IOC_WRITE means userland is writing and kernel is
 * reading. _IOC_READ means userland is reading and kernel is writing.
 */
// 无数据传输
#ifndef _IOC_NONE
# define _IOC_NONE	0U
#endif
// 用户空间写入
#ifndef _IOC_WRITE
# define _IOC_WRITE	1U
#endif
// 用户空间读取
#ifndef _IOC_READ
# define _IOC_READ	2U
#endif
```

内核定义了_IO(), _IOR(), _IOW(), IOWR()这4个宏用来辅助生成cmd
```c
#define _IOC(dir,type,nr,size) \
	(((dir)  << _IOC_DIRSHIFT) | \
	 ((type) << _IOC_TYPESHIFT) | \
	 ((nr)   << _IOC_NRSHIFT) | \
	 ((size) << _IOC_SIZESHIFT))

/*
 * Used to create numbers.
 *
 * NOTE: _IOW means userland is writing and kernel is reading. _IOR
 * means userland is reading and kernel is writing.
 */
#define _IO(type,nr)		_IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)	_IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOW(type,nr,size)	_IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOWR(type,nr,size)	_IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
```

解析cmd命令
```c
/* used to decode ioctl numbers.. */
#define _IOC_DIR(nr)		(((nr) >> _IOC_DIRSHIFT) & _IOC_DIRMASK)
#define _IOC_TYPE(nr)		(((nr) >> _IOC_TYPESHIFT) & _IOC_TYPEMASK)
#define _IOC_NR(nr)		(((nr) >> _IOC_NRSHIFT) & _IOC_NRMASK)
#define _IOC_SIZE(nr)		(((nr) >> _IOC_SIZESHIFT) & _IOC_SIZEMASK)
```

魔数：范围为0-255，通常用英文字符'A'-'Z'或'a'-'z'来表示，设备驱动程序从传递进来的命令获取魔数，然后与自身处理的魔数相比较，如果相同则处理，不同则不处理。

序列号：范围为0-255

变量类型：传入变量类型的大小