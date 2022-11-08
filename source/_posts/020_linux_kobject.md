---
title: Linux设备模型 kobject
author: Jack
tags:
  - Linux
  - Platform
categories:
  - Linux
abbrlink: f8def83d
---

+ Linux内核中有大量的驱动，而这些驱动往往具有类似的结构，根据面向对象的思想，可以将共同的部分提取为父类，这个父类就是`kobject`，`kobject`中包含了大量设备的必须信息，三大类设备驱动都需要包含这个`kobject`结构，从面向对象的思想来看，即继承自`kobject`，一个`kobject`对象往往就对应sysfs中的一个目录，`kobject`是组成设备模型的基本结构，`kobject`需要处理的基本任务如下：
    + 对象的引用计数，当一个内核对象被创建时，不知道该对象的存活时间，跟踪该对象的生命周期的一个方法就是使用引用计数，
    + sysfs表述

+ `kobject`的结构体定义
    ```c
    struct kobject {
        const char		*name;
        struct list_head	entry;
        struct kobject		*parent;
        struct kset		*kset;
        struct kobj_type	*ktype;
        struct kernfs_node	*sd; /* sysfs directory entry */
        struct kref		kref;
    #ifdef CONFIG_DEBUG_KOBJECT_RELEASE
        struct delayed_work	release;
    #endif
        unsigned int state_initialized:1;
        unsigned int state_in_sysfs:1;
        unsigned int state_add_uevent_sent:1;
        unsigned int state_remove_uevent_sent:1;
        unsigned int uevent_suppress:1;
    }
    ```
    + name: 表示`kobject`对象的名字，对应sysfs下的一个目录
    + entry: `list_head`链表，用来链接各个`kobject`对象
    + parent: 指向当前`kobject`父对象的指针
    + kset: 表示当前`kobject`所属的集合
    + ktype: 表示当前`kobject`的类型
    + sd: 表示VFS文件系统的目录项
    + kref: 为`kobject`的引用计数，当引用计数为0时，就回调`release`方法释放该对象
    + state_initialized:1: 初始化标志位，在对象初始化时被置位，表示对象是否被初始化
    + state_in_sysfs:1: 表示`kobject`在sysfs中的状态，在对应目录中被创建则为1，否则为0
    + state_add_uevent_sent:1: 添加设备的uevent事件是否被发送标志
    + state_remove_uevent_sent:1: 删除设备的uevent事件是否被发送标志
    + 