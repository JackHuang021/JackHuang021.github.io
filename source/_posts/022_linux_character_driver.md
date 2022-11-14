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
    + `loff_t (*llseak) (struct file *, loff_t, int)`：该函数用于修改文件当前的读写位置
    + `ssize_t (*read) (struct file *, char __user *, size_t, loff_t *)`：用于读取设备文件
    + `ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *)`：用于向设备文件写入数据
    + `unsigned int (*poll) (struct file *, struct poll_table_struct *)`：用于查询设备是否可以进行非阻塞的读写
    + `long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long)`：与应用程序中的`ioctl`函数对应
    + `long (*compat_ioctl) (struct file *, unsigned int, unsigned long)`：功能与`unlocked_ioctl()`函数功能一样，区别在于在64位系统上，32位的应用程序调用将会使用此函数，在32位的系统上运行32位的应用程序调用的是`unlocked_ioctl()`
    + `int (*mmap) (struct file *, struct vm_area_struct *)`：用于将设备的内存映射到进程空间（即用户空间），一般帧缓冲设备会使用此函数，比如LCD驱动的显存，将帧缓冲映射到用户空间中以后应用程序就可以直接操作显存了，这样就不用在用户空间和内核空间之间来回复制
    + `int (*open) (struct inode *, struct file *)`、：用于打开设备文件
    + `int (*release) (struct inode *, struct file *)`：用于释放设备文件，与应用程序中的`close()`函数对应
    + `int (*fsync) (int, struct file *, int)`：用于刷新待处理的数据，将缓冲区的数据刷新到磁盘中

#### sysfs文件系统目录结构
+ `sysfs`文件系统提供了一种用户与内核数据结构进行交互的方式，Linux设备模型中，设备、驱动、总线组织成拓扑结构，通过`sysfs`文件系统以目录结构进行展示与管理
+ `devices`：内核对系统中所有设备的分层次表达模型，也是`/sys`文件系统管理设备的最重要的目录结构
+ `dev`：这个目录下维护一个按字符设备和块设备的设备号文件(major:minor)链接到真实的设备
+ `bus`：内核按照总线类型分层放置的目录结构，`devices`中的所有设备都是连接于某种总线之下，每一种具体总线之下可以找到每一个具体设备的符号链接，是构成Linux统一设备模型的一部分
+ `class`：这是按照设备功能分类的设备模型，如系统所有输入设备都会出现在`/sys/class/input`之下，不论它们是以何种总线连接到系统，它也是构成Linux统一设备模型的一部分
+ `block`：这是系统中当前所有块设备所在的目录，按照功能来说放置在`/sys/class`下更恰当，由于历史遗留因素而一直存在于`/sys/block`，在2.6.26内核中已经正式移到`/sys/class/block`，`/sys/block`中的内容已经变为指向它们在`/sys/devices`中真实设备的符号链接文件
+ `firmware`：系统加载固件机制的对用户空间的接口
+ `fs`：用于描述系统中所有文件系统，包括文件系统本身和按文件系统分类存放的已挂载点
+ `kernel`：内核所有可调整参数的位置，目前只有`uevent_helper`,`kexec_loaded`,`mm`和新式的`slab`分配器等几项较新的设计在使用，其它内核可调整参数仍然位于`/proc/sys/kernel`接口中
+ `module`：系统中所有模块的信息，不论这些模块是以内联方式编译到内核镜像`vmlinux`中还是编译为外部

#### 对字符设备的封装：`cdev`结构体
+ `cdev`结构体定义在`include/linux/cdev.h`中
    ```c
    struct cdev {
        struct kobject kobj;
        struct module *owner;
        const struct file_operations *ops;
        struct list_head list;
        dev_t dev;
        unsigned int count;
    } __randomize_layout;
    ```
+ 其中[`__randomsize_layout`](https://lwn.net/Articles/722293/)表示随机化一个结构体，在编译时随机排布结构体中元素的顺序，从而使攻击者无法通过地址偏移的方式获取一些敏感数据成员的信息
+ `struct kobject kobj;`：抽象出来的用来表示设备模型的数据结构
+ `const struct file_operations *ops;`：定义了字符设备驱动提供给虚拟文件系统的接口函数集合
+ `struct list_head list;`：将所有字符设备通过链表进行管理
+ `dev_t dev`：设备号，`dev_t`数据结构定义在`include/linux/types.h`中，包含主设备号和次设备号，实际是一个32位整形数据，其中高12位为主设备号，低20位为次设备号
    ```c
    // dev_t的定义
    typedef u32 __kernel_dev_t;
    typedef __kernel_dev_t		dev_t;

    // 设备号的一些操作宏
    #define MINORBITS	20
    #define MINORMASK	((1U << MINORBITS) - 1)

    #define MAJOR(dev)	((unsigned int) ((dev) >> MINORBITS))
    #define MINOR(dev)	((unsigned int) ((dev) & MINORMASK))
    #define MKDEV(ma,mi)	(((ma) << MINORBITS) | (mi))
    ```
+ `unsigned int count;`：属于同主设备号的次设备号的个数

#### 设备管理机制：`kobject`
+ `kobject`是Linux设备驱动模型的基础，也是设备模型中抽象的一部分，它与sysfs文件系统紧密相连，在内核中注册的每个`kobject`对象对应sysfs文件系统中的一个目录
+ `kobject`是组成设备模型的基本结构，是所有用来描述设备模型的数据结构的基类，它嵌入在所有的描述设备模型的容器对象中，例如bus,devices,drivers，这些容器通过`kobject`链接起来，形成一个树形结构
+ 该结构体定义在`include/linux/kobject.h`中，通过这个数据结构可以使所有设备在底层都具有统一的接口，`kobject`提供基本的对象管理，结构体内容如下：
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
    };
    ```
+ `const char *name;`：`kobject`名字，对应`sysfs`下的一个目录
+ `struct list_head entry;`：`kobject`中插入的的`list_head`链表结构，用于构造双向链表
+ `struct kobject *parent;`：指向当前`kobject`父对象的指针，体现在`sysfs`中就是包含当前`kobject`对象的目录对象
+ `struct kset *kset;`：当前`kobject`对象所属的集合
+ `struct kobj_type *ktype;`：当前`kobject`对象的类型
+ `struct kernfs_node *sd;`：VFS文件系统的目录项，是设备和文件之间的桥梁，`sysfs`中的符号链接是通过`kernel_node`内的联合体实现的
+ `struct kref kref;`：`kobject`的引用计数，当计数为0时，回调之前注册的`release`方法释放对象
+ `unsigned int state_initialized:1;`：初始化标志位，初始化时被置位
+ `unsigned int state_in_sysfs:1;`：`kobject`在`sysfs`中的状态，在目录中创建则为1，否则为0
+ `unsigned int state_add_uevent_sent:1`：添加设备的`uevent`事件是否发送标志，添加设备时向用户控件发送`uevent`事件，请求新增设备
+ `unsigned int state_remove_uevent_sent:1`：删除设备的`uevent`事件是否发送标志，删除设备时向用户控件发送`uevent`事件，请求卸载设备
+ `unsigned int uevent_suppress:1`：是否忽略上报`uevent`

#### `kobject`集合 `kset`
+ `kset`是包含多个`kobject`的集合，如果在`sysfs`的目录中包含多个子目录，那需要将它定义成一个`kset`
    ```c
    /**
     * struct kset - a set of kobjects of a specific type, belonging to a specific subsystem.
     *
     * A kset defines a group of kobjects.  They can be individually
     * different "types" but overall these kobjects all want to be grouped
     * together and operated on in the same manner.  ksets are used to
     * define the attribute callbacks and other common events that happen to
     * a kobject.
     *
     * @list: the list of all kobjects for this kset
     * @list_lock: a lock for iterating over the kobjects
     * @kobj: the embedded kobject for this kset (recursion, isn't it fun...)
     * @uevent_ops: the set of uevent operations for this kset.  These are
     * called whenever a kobject has something happen to it so that the kset
     * can add new environment variables, or filter out the uevents if so
     * desired.
     */
    struct kset {
        struct list_head list;
        spinlock_t list_lock;
        struct kobject kobj;
        const struct kset_uevent_ops *uevent_ops;
    };
    ```
+ `struct list_head list;`：包含在`kset`在内的所有`kobject`构成的一个双向链表
+ `spinlock_t list_lock;`：遍历`kobject`时的锁
+ `struct kobject kobj;`：归属于该`kset`的所有的`kobject`的parent
+ `const struct kset_uevent_ops *uevent_ops;`：`kset`的`uevent`操作函数集，当`kset`中的`kobject`有状态变化时，会回调这个函数集


#### `kobject`类型 `ktype`
+ `kobj_type`用于表征`kobject`的类型，指定了删除`kobject`时要调用的函数，`kobject`结构体中有`struct kref`字段对`kobject`进行引用计数，当计数值为0时，就会调用`kobj_type`中的`release`函数对`kobject`进行释放
+ `kobj_type`指定了通过`sysfs`显示或修改有关`kobject`的信息时要处理的操作，实际是调用`show/store`函数
    ```c
    struct kobj_type {
        void (*release)(struct kobject *kobj);
        const struct sysfs_ops *sysfs_ops;
        struct attribute **default_attrs;
        const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
        const void *(*namespace)(struct kobject *kobj);
    };

    struct sysfs_ops {
        ssize_t	(*show)(struct kobject *, struct attribute *, char *);
        ssize_t	(*store)(struct kobject *, struct attribute *, const char *, size_t);
    }

    struct attribute {
        const char		*name;
        umode_t			mode;
    #ifdef CONFIG_DEBUG_LOCK_ALLOC
        bool			ignore_lockdep:1;
        struct lock_class_key	*key;
        struct lock_class_key	skey;
    #endif
    };
    ```
+ `void (*release)(struct koject *kobj);` 释放`kobject`对象的接口，有点类似面向对象中的析构
+ `const struct sysfs_ops *sysfs_ops;` 操作`kobject`的方法集
+ `struct attribute **default_attrs;` 所谓的`attribute`就是内核控件和用户空间进行信息交互的一种方法，例如某个driver定义了一个变量，却希望用户空间程序可以修改该变量，那么可以将该变量以`sysfs attribute`的形式开放出来









