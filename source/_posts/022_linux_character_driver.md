---
title: Linux 字符设备驱动开发
author: Jack
tags:
- Linux
- Driver
categories: 
- Linux
---

#### 字符设备驱动简介

+ 字符设备是Linux驱动中最基本的一类设备驱动，字符设备就是一个一个字节，按照字节流进行读写操作的设备，读写数据是分先后顺序的，这些设备的驱动就叫做字符设备驱动

+ 应用程序运行在用户空间，Linux驱动属于内核的一部分，因此驱动运行于内核空间，当在用户空间想要实现对内核的操作，比如使用`open`函数打开`/dev/led`这个设备，因为用户空间不能直接对内核进行操作，因此必须使用一个叫做“系统调用”的方法来实现从用户空间陷入到内核空间，这样才能实现对底层驱动的操作。

+ 应用程序调用`open()`函数的流程：应用调用open函数（应用程序） -> C库中的open（）函数 -> open（）系统调用

+ 每一个系统调用在驱动中都有与之对应的一个驱动函数，在Linux内核文件`include/linux/fs.h`中有一个叫做`file_operations`的结构体，此结构体就是Linux内核驱动操作函数集合，该结构体内容如下：
    ```c
    struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
        int (*iterate) (struct file *, struct dir_context *);
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        int (*mremap)(struct file *, struct vm_area_struct *);
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *, fl_owner_t id);
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, loff_t, loff_t, int datasync);
        int (*aio_fsync) (struct kiocb *, int datasync);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
        unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
        int (*check_flags)(int);
        int (*flock) (struct file *, int, struct file_lock *);
        ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
        ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
        int (*setlease)(struct file *, long, struct file_lock **, void **);
        long (*fallocate)(struct file *file, int mode, loff_t offset,
                  loff_t len);
        void (*show_fdinfo)(struct seq_file *m, struct file *f);
    #ifndef CONFIG_MMU
        unsigned (*mmap_capabilities)(struct file *);
    #endif
    };
    ```
    + `owner`拥有该结构体的模块的指针，一般设置为`THIS_MODULE`
    + `loff_t (*llseak) (struct file *, loff_t, int)`，该函数用于修改文件当前的读写位置
    + `ssize_t (*read) (struct file *, char __user *, size_t, loff_t *)`，用于读取设备文件
    + `ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *)`，用于向设备文件写入数据
    + `unsigned int (*poll) (struct file *, struct poll_table_struct *)`，用于查询设备是否可以进行非阻塞的读写
    + `long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long)`，与应用程序中的`ioctl`函数对应
    + `long (*compat_ioctl) (struct file *, unsigned int, unsigned long)`,功能与`unlocked_ioctl()`函数功能一样，区别在于在64位系统上，32位的应用程序调用将会使用此函数，在32位的系统上运行32位的应用程序调用的是`unlocked_ioctl()`
    + `int (*mmap) (struct file *, struct vm_area_struct *)`，用于将设备的内存映射到进程空间（即用户空间），一般帧缓冲设备会使用此函数，比如LCD驱动的显存，将帧缓冲映射到用户空间中以后应用程序就可以直接操作显存了，这样就不用在用户空间和内核空间之间来回复制
    + `int (*open) (struct inode *, struct file *)`，用于打开设备文件
    + `int (*release) (struct inode *, struct file *)`，用于释放设备文件，与应用程序中的`close()`函数对应
    + `int (*fsync) (int, struct file *, int)`，用于刷新待处理的数据，将缓冲区的数据刷新到磁盘中

#### 驱动的加载和卸载
+ 从源码编译完内核后

