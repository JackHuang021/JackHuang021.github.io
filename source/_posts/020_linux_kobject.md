---
title: Linux设备模型
author: Jack
tags:
  - Linux
  - Platform
categories:
  - Linux
abbrlink: f8def83d
---

#### 背景
为了降低设备多样性带来的Linux驱动开发的复杂度，以及设备热插拔处理，电源管理等，Linux内核提出了设备模型（Driver Model）的概念。设备模型将硬件设备归纳、分类，然后抽象出一套标准的数据结构和接口，驱动的开发，就简化为对内核所规定的数据结构的填充和实现

<!-- more -->

#### 设备模型基本概念
##### Linux设备模型组成
硬件拓扑描述Linux设备模型中四个重要概念：Bus、Class、Device、Device Driver
1. Bus：总线是CPU和一个或多个设备之间信息交互的通道，而为了方便设备模型的抽象，所有的设备都应连接到总线上
2. Class：在Linux设备模型中，Class的概念非常类似面向对象程序设计中的Class，它主要是集合具有相似功能或属性的设备，这样就可以抽象出一套可以在多个设备之间共用的数据结构和接口函数，因为从属于相同Class的设备和驱动程序就不需要重复定义这些公共资源，直接从Class中继承
3. Device：抽象系统中所有的硬件设备，描述它的名字、属性、从属的Bus、从属的Class
4. Device Driver：Linux设备模型用Driver抽象硬件设备的驱动程序，它包含设备初始化、电源管理相关的接口实现，Linux内核中的驱动开发，基本都围绕该抽象进行实现

##### 设备模型的核心思想
Linux设备模型的核心思想
1. 用Device和Device Driver两个数据结构，分别从“有什么用”和“怎么用”两个角度描述硬件设备，统一了编写设备驱动的格式
2. 通过`Bus->Device`类型的树状结构解决设备之间的依赖，启动某一个设备前，内核会检查该设备是否依赖其它设备或者总线，如果依赖，则检查所依赖的对象是否已经启动，如果没有，则会先启动它们，直到启动该设备的条件具备为止。而驱动开发人员需要做的，就是在编写设备驱动时，告知内核该设备的依赖关系即可
3. 使用Class结构，在设备模型中引入面向对象的概念，这样可以最大限度地抽象共性，减少驱动开发过程中的重复劳动，降低工作量

#### KObject
Linux内核中有大量的驱动，而这些驱动往往具有类似的结构，根据面向对象的思想，可以将共同的部分提取为父类，这个父类就是`kobject`，`kobject`中包含了大量设备的必须信息，三大类设备驱动都需要包含这个`kobject`结构，从面向对象的思想来看，即继承自`kobject`，一个`kobject`对象往往就对应sysfs中的一个目录，`kobject`是组成设备模型的基本结构，`kobject`需要处理的基本任务如下：

+ 通过parent指针，可以将所有kobject以层次结构的形式组合起来
+ 对象的引用计数，当一个内核对象被创建时，不知道该对象的存活时间，跟踪该对象的生命周期的一个方法就是使用引用计数，当内核中没有代码持有该对象的引用时，说明该对象可以被销毁了
+ 和sysfs虚拟文件系统配合，将每一个kobject及其特性，以文件的形式开放到用户空间

*注1：在Linux中，kobject几乎不会单独存在，它的主要功能就是内嵌在一个大型的数据结构中，为这个数据结构提供一些底层的功能实现*

*注2：Linux driver开发者，很少会直接使用Kobject以及它提供的接口，而是使用构建在Kobject之上的设备模型接口*

kobject, kset, ktype三者的概念
+ kobject是基本数据类型，每个kobject都会在`/sys`文件系统中以目录的形式出现
+ ktype代表kobject的属性操作集合，
+ Kset是一个特殊的Kobject（因此它也会在`/sys`文件系统中以目录的形式出现），它用来集合相似的Kobject

`kobject`的结构体定义
```c
// include/linux/kobject.h
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
+ name: 表示`kobject`对象的名字，对应sysfs下的一个目录
+ entry: `list_head`链表，用来链接各个`kobject`对象
+ parent: 指向当前`kobject`父对象的指针
+ kset: 表示当前`kobject`所属的集合
+ ktype: 表示当前`kobject`的类型
+ sd: 在sysfs中的表示
+ kref: 为`kobject`的引用计数，当引用计数为0时，就回调`release`方法释放该对象
+ state_initialized: 初始化标志位，在对象初始化时被置位，表示对象是否被初始化
+ state_in_sysfs: 表示`kobject`在sysfs中的状态，在对应目录中被创建则为1，否则为0
+ state_add_uevent_sent: 添加设备的uevent事件是否被发送标志
+ state_remove_uevent_sent: 删除设备的uevent事件是否被发送标志
+ uevent_suppress：如果该字段为1，则表示忽略所有上报的uevent事件

*uevent提供了“用户空间通知”的功能实现，通过该功能，当内核中有kobject的增加、删除、修改等动作时，会通知用户空间*

kset的结构体定义
```c
//include/linux/kobject.h
struct kset {
    struct list_head list;
    spinlock_t list_lock;
    struct kobject kobj;
    const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```
+ list, list_lock：用于保存该kset下所有的kobject链表
+ kobj，该kset自己的kobject（kset是一个特殊的kobject，也会在sysfs中以目录的形式体现）
+ uevent_ops：该kset的uevent操作函数集。当任何Kobject需要上报uevent时，都要调用它所从属的kset的uevent_ops，添加环境变量，或者过滤event（kset可以决定哪些event可以上报）。因此，如果一个kobject不属于任何kset时，是不允许发送uevent的

ktype的结构体定义
```c
//include/linux/ktype.h
struct kobj_type {
    void (*release)(struct kobject *kobj);
    const struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
    const struct attribute_group **default_groups;
    const struct kobj_ns_type_operations *(*child_ns_type)（struct kobject *kobj）;
    const void *(*namespace)(struct kobject *kobj);
    void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```
+ release: 通过该回调函数，可以将包含该种类型kobject的数据结构的内存空间释放掉
+ sysfs_ops：该种类型的kobject的sysfs文件系统接口
+ default_attrs：该种类型的kobject的atrribute列表（所谓attribute，就是sysfs文件系统中的一个文件）。将会在Kobject添加到内核时，一并注册到sysfs中
+ child_ns_type, namespace：和文件系统（sysfs）的命名空间有关

每一个内嵌kobject的数据结构，例如kset、device、device_driver等等，都要实现一个ktype，并定义其中的回调函数，sysfs的相关操作也一样，必须经过ktype的中转，因为sysfs看到的是kobject，而真正的文件操作的主体，是内嵌kobject的上层数据结构

##### kobject使用流程
kobject在大多数情况下会嵌在其他数据结构中使用，使用流程如下：
1. 定义一个`struct kset`类型的指针，并在初始化时为它分配空间，添加到内核中
2. 根据实际情况，定义自己所需的数据结构原型，该数据结构中包含有kobjet
3. 定义一个适合自己的ktype，并实现其中回调函数
4. 在需要用到包含kobject的数据结构中，动态分配该数据结构，并分配kobject空间，添加到内核中
5. 每一次引用数据结构时，调用`kobject_get`接口增加引用计数；引用结束时，调用`kobject_put`接口，减少引用计数
6. 当引用计数减少为0时，kobject模块调用ktype所提供的release接口，释放上层数据结构以及kobject的内存空间

#### Uevent
##### Uevent的功能
Uevent是kobject的一部分，用于在kobject状态发生改变时，例如增加、移除等，通知用户空间程序，用户空间程序收到这样的事件后，会做相应的处理

该机制通常是用来支持热拔插设备的，例如U盘插入后，USB相关的驱动软件会动态创建用于表示该U盘的device结构（相应的也包括其中的kobject），并告知用户空间程序，为该U盘动态的创建/dev/目录下的设备节点，更进一步，可以通知其它的应用程序，将该U盘设备mount到系统中，从而动态的支持该设备。

##### Uevent代码实现

```c
// include/linux/kobject.h
enum kobject_action {
    KOBJ_ADD,
    KOBJ_REMOVE,
    KOBJ_CHANGE,
    KOBJ_MOVE,
    KOBJ_ONLINE,
    KOBJ_OFFLINE,
    KOBJ_BIND,
    KOBJ_UNBIND,
};
```
+ ADD/REMOVE: kobject（或上层数据结构）的添加、移除事件
+ ONLINE/OFFLINE： kobject（或上层数据结构）的上线、下线事件
+ CHANGE：Kobject（或上层数据结构）的状态或者内容发生改变
+ MOVE：Kobject（或上层数据结构）更改名称或者更改Parent（意味着在sysfs中更改了目录结构）
+ CHANGE：如果设备驱动需要上报的事件不再上面事件的范围内，或者是自定义的事件，可以使用该event，并携带相应的参数

在利用kmod向用户空间上报event事件时，会直接执行用户空间的可执行文件，而在Linux系统，可执行文件的执行依赖于环境变量，因此`kobj_uevent_env`用于组织此次事件上报时的环境变量
```c
// include/linux/kobject.h
#define UEVENT_NUM_ENVP         64
#define UEVENT_BUFFER_SIZE      2048

struct kobj_uevent_env {
    char *argv[3];
    char *envp[UEVENT_NUM_ENVP];
    int envp_idx;
    char buf[UEVENT_BUFFER_SIZE];
    int buflen;
};
```
+ envp：指针数组，用于保存每个环境变量的地址，最多可支持的环境变量数量为UEVENT_NUM_ENVP
+ envp_idx：用于访问环境变量指针数组的index
+ buf：保存环境变量的buffer，最大为UEVENT_BUFFER_SIZE
+ buflen：访问buf的变量

```c
struct kset_uevent_ops {
    int (* const filter)(struct kset *kset, struct kobject *kobj);
    const char *(* const name)(struct kset *kset, struct kobject *kobj);
    int (* const uevent)(struct kset *kset, struct kobject *kobj,
                        struct kobj_uevent_env *env);
};
```
`kset_uevent_ops`是为kset定制的一个数据结构，里面包含filter和uevent两个回调函数，用处如下：
+ filter：当任何kobject需要上报uevent时，它所属的kset可以通过该接口过滤，阻止不希望上报的event，从而达到从整体上管理的目的
+ name：该接口可以返回kset的名称，如果一个kset没有合法的名称，则其下的所有kobject将不允许上报uevent
+ uevent：当任何kobject需要上报uevent时，它所属的kset可以通过给接口统一为这些event添加环境变量，因为很多时候上报uevent时的环境变量都是相同的，因此可以由kset统一处理，就不需要让每个Kobject独自添加了

#### sysfs
##### sysfs介绍
sysfs是一个基于RAM的文件系统，他和kobject一起，可以将kernel的数据结构导出到用户空间，以文件目录结构的形式，提供对这些数据结构的访问支持

##### sysfs和kobject的关系
每一个kobject都会对应sysfs中的一个目录，因此在将kobject添加到kernel时，`create_dir`接口会调用sysfs文件系统的创建目录接口，创建和kobject对应的目录

##### attribute功能
attribute是对应kobject而言的，指的是kobject的属性，sysfs中的目录描述了kobject，而kobject是特定数据结构类型变量（如struct device）的体现，因此kobject的属性，就是这些变量的属性，它可以是任何东西，名称、内部变量、字符串等，而attribute在sysfs文件系统中是以文件的形式提供的

attribute就是内核空间和用户空间进行信息交互的一种方法，例如某个driver定义了一个变量，却希望用户空间程序可以修改这个变量，以控制driver的运行行为，那么就可以将该变量以sysfs attribute的形式开放出来

##### attribute定义
Linux内核中，attribute分为普通的attribute和二进制attribute，定义如下:
```c
// include/linux/types.h
typedef unsigned short umode_t;

// include/linux/sysfs.h 
struct attribute {
    const char *name;
    umode_t mode;
};

// include/linux/sysfs.h 
struct bin_attribute {
    struct attribute attr;
    size_t size;
    void  *private;
    ssize_t (*read)(struct file *, struct kobject *,
                struct bin_attribute *, char *, loff_t, size_t);
    ssize_t (*write)(struct file *, struct kobject *,
                struct bin_attribute );
    int (*mmap)(struct file *, struct kobject *, 
                struct bin_attribute *attr, struct vm_area_struct *vma);
};
```
`struct attribute`为普通的attribute，使用该attribute生成的sysfs文件，只能用字符串的形式读写，而`struct bin_attribute`在`struct attribute`的基础上增加了read、write等函数，因此它所生成的sysfs文件可以用任何方式读写

#### device和device driver
##### 关键数据结构
struct device数据结构
```c
struct device {
	struct kobject kobj;
	struct device		*parent;

	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */
#ifdef CONFIG_PROVE_LOCKING
	struct mutex		lockdep_mutex;
#endif
	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

#ifdef CONFIG_ENERGY_MODEL
	struct em_perf_domain	*em_pd;
#endif

#ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
	struct irq_domain	*msi_domain;
#endif
#ifdef CONFIG_PINCTRL
	struct dev_pin_info	*pins;
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
	struct list_head	msi_list;
#endif
#ifdef CONFIG_DMA_OPS
	const struct dma_map_ops *dma_ops;
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

#ifdef CONFIG_DMA_DECLARE_COHERENT
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */
#endif
#ifdef CONFIG_DMA_CMA
	struct cma *cma_area;		/* contiguous memory area for dma
					   allocations */
#endif
	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	bool			dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
	bool			dma_ops_bypass : 1;
#endif
};
```
+ parent: 该设备的父设备，一般是该设备所从属的bus controller等设备
+ p：一个用于struct device的私有数据结构指针，该指针中保存子设备链表，用于添加到bus/driver/parent等设备中的链表头等
+ kobj：该数据结构对应的kobject
+ init_name：设备名称，在设备模型中，名称是一个非常重要的变量，任何注册到内核中的设备，都必须有一个合法的名称，可以在初始化时给出，也可以由内核根据"bus_name + device id"的方式去创造
+ type：`struct device_type`它和`struct device`的关系非常类似`struct kobject`和`struct kobj_type`之间的关系
+ bus：该device属于哪个总线
+ driver：该device对应的device driver
+ platform_data：用于保存具体的平台相关的数据，可以将一些私有的数据暂存在这里，需要使用的时候，再拿出来
+ driver_data：用于保存和driver相关的私有数据
+ dev_t：一个32位的整数，由（major和minor）组成，以设备节点的形式向用户空间提供接口的设备中，当作设备号使用，该变量主要用于在sys文件系统中，为每个具有设备号的device，创建`/sys/dev/*`下的对应目录
+ class：该设备属于哪个class
+ group：该设备的默认attribute集合，将会在设备注册时自动在sysfs中创建对应的文件

struct device_driver数据结构
```c
// include/linux/device/driver.h
struct device_driver {
    const char *name;
    struct bus_type *bus;
    
    struct module *owner;
    const char *mod_name;

    bool suppress_bind_attrs;
    enum probe_type probe_type;

    const struct of_device_id *of_match_table;
    const struct acpi_device_id *acpi_match_table;

    int (*probe) (struct devcie *dev);
    void (*sync_state)(struct device *dev);
    int (*remove) (struct device *dev);
    void (*shutdown) (struct device *dev);
    int (*suspend) (struct device *dev, pm_message_t state);
    int (*resume) (struct device *dev);
    const struct attribute_group **groups;
    const struct attribute_group **dev_groups;

    const struct dev_pm_ops *pm;
    void (*coredump) (struct device *dev);
    
    struct drvier_private *p;
}
```
+ name：driver名称，和device结构一样，名称非常重要
+ bus：该driver所驱动设备的bus，内核要保证在driver运行前，设备所依赖的总线正确初始化
+ owner, mod_name：内核module相关的变量
+ suppres_bind_attrs：是否在sysfs中启用bind和unbind attribute
+ probe, remove：这两个接口函数用于实现driver逻辑的开始和结束，在设备模型的结构下，只有driver和device同时存在时，才需要开始执行driver的代码逻辑。这也是probe和remove两个接口名称的由来：检测到了设备和移除了设备
+ shutdown、suspend、resume、pm：电源管理相关的内容
+ groups：和struct device结构中的同名变量类似，driver也可以定义一些默认attribute，这样在将driver注册到内核中时，内核设备模型部分的代码（driver/base/driver.c）会自动将这些attribute添加到sysfs中
+ p：driver的私有数据指针，其他模块不能访问

##### 设备模型框架下驱动开发的基本步骤
在设备模型框架下，设备驱动的开发主要包括2个步骤
1. 分配一个`struct device`类型的变量，填充必要的信息后，把它注册到内核中
2. 分配一个`struct device_driver`类型的变量，填充必要的信息后，把它注册到内核中

这两步完成后，内核会在合适的时机，调用`struct device_driver`变量中的probe, remove, suspend, resume等回调函数，从而触发或终结设备驱动的执行，所有的驱动程序逻辑都会由这些回调函数实现

*一般情况下，Linux驱动开发很少直接使用device和device_driver，因为内核在它们之上又封装了一层，如platform_device等，这些层次提供的接口更为简单，易用*


#### Bus
在Linux设备模型中，Bus（总线）是一类特殊的设备，它是连接处理器和其它设备之间的通道（channel）。为了方便设备模型的实现，内核规定，系统中的每个设备都要连接在一个Bus上，这个Bus可以是一个内部Bus、虚拟Bus或者Platform Bus

##### bus定义
内核通过`struct bus_type`结构抽象bus
```c
// include/linux/device/bus.h
struct bus_type {
    const char *name;
    const char *dev_name;
    struct device *dev_root;
    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;

    int (*match)(struct device *dev, struct device_driver *drv);
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    void (*sync_state)(struct device *dev);
    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);

    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    int (*num_vf)(struct device *dev);

    int (*dma_configure)(struct device *dev);

    const struct dev_pm_ops *pm;
    const struct iommu_ops *iommu_ops;

    struct subsys_private *p;
    struct lock_class_key lock_key;

    bool need_parent_lock;
};
```
+ name: 该bus的名称，会在sysfs中以目录的形式存在，platform bus在sysfs中表现为"/sys/bus/platform"
+ dev_name: 该名称和`struct device`结构中的init_name有关，对有些设备而言（例如批量化的USB设备），设计者根本就懒得为它起名字的，而内核也支持这种懒惰，允许将设备的名字留空。这样当设备注册到内核后，设备模型的核心逻辑就会用"bus->dev_name+device ID”的形式，为这样的设备生成一个名称
+ dev_root：和sub system功能有关
+ bus_groups, dev_groups, drv_groups：一些默认的attribute组
+ match：一个由具体的bus driver实现的回调函数，当任何属于该bus的device或device_driver添加到内核时，内核都会调用该接口，如果新加的device或device_driver匹配上了自己的另一半的话，该接口返回非零值，此时bus模块的核心逻辑就会执行后续的处理
+ uevent：一个由具体的bus driver实现的回调函数，当任何属于该Bus的device，发生添加、移除或者其它动作时，bus模块的核心逻辑就会调用该接口，以便bus driver能够修改环境变量
+ probe, remove：这两个回调函数，和device_driver中的非常类似，如果probe指定的device的话，需要保证该device所在的bus是被初始化过、确保能正确工作的，这就需要在执行device_driver的probe前，先执行它的bus的probe
+ shutdown, suspend, resume：和probe, remove的原理类似，这些是电源管理相关的实现

subsys_private定义，subsys_private可以看做是bus的私有数据
```c
// drivers/base/base.h 
struct subsys_private {
    struct kset subsys;
    struct kset *devices_kset;
    struct list_head interfaces;
    struct mutex mutex;

    struct kset *drivers_kset;
    struct klist klist_devices;
    struct klist klist_drivers;
    struct blocking_notifier_head bus_notifier;
    unsigned int drivers_autoprobe:1;
    struct bus_type *bus;
    struct kset glue_dirs;
    struct class *class;
};
```
+ subsys, devices_kset, drivers_kset：subsys代表了本bus的kobject集合，它下面可以包含其它的kset或者其它的kobject，devices_kset和drivers_kset则是bus下面的两个kset（如/sys/bus/spi/devices和/sys/bus/spi/drivers），分别包括本bus下所有的device和device_driver
+ interfaces：一个list_head链表，用于保存该bus下所有的interface
+ klist_devices和klist_drivers：这两个链表分别保存了本bus下所有的device和device_driver指针，以方便查找
+ drivers_autoprobe：用于控制该bus下的drivers或者devices是否自动probe
+ bus, class：分别保存上层的bus和class指针

bus模块的功能包括：
+ bus的注册和注销
+ 本bus下device或者device_driver注册或注销到内核的处理
+ device_driver的probe处理
+ 管理bus下的所有device和device_driver

bus的注册是由`bus_register()`接口实现的，在接口的原型在`include/linux/device.h`中声明，在`drivers/base/bus.c`中实现
```c
// drivers/base/bus.c
/**
 * bus_register - register a driver-core subsystem
 * @bus: bus to register
 *
 * Once we have that, we register the bus with the kobject
 * infrastructure, then register the children subsystems it has:
 * the devices and drivers that belong to the subsystem.
 */
int bus_register(struct bus_type *bus)
{
	int retval;
	struct subsys_private *priv;
	struct lock_class_key *key = &bus->lock_key;

    // 为subsys_private指针分配空间
	priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

    // 更新priv->bus和bus->p的值
	priv->bus = bus;
	bus->p = priv;

	BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

	retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
	if (retval)
		goto out;

	priv->subsys.kobj.kset = bus_kset;
	priv->subsys.kobj.ktype = &bus_ktype;
	priv->drivers_autoprobe = 1;

	retval = kset_register(&priv->subsys);
	if (retval)
		goto out;

    // 向bus目录下添加一个uevent attribute
	retval = bus_create_file(bus, &bus_attr_uevent);
	if (retval)
		goto bus_uevent_fail;

    // 向内核添加devices kset和drivers kset，并在sysfs中添加目录
	priv->devices_kset = kset_create_and_add("devices", NULL,
						 &priv->subsys.kobj);
	if (!priv->devices_kset) {
		retval = -ENOMEM;
		goto bus_devices_fail;
	}

	priv->drivers_kset = kset_create_and_add("drivers", NULL,
						 &priv->subsys.kobj);
	if (!priv->drivers_kset) {
		retval = -ENOMEM;
		goto bus_drivers_fail;
	}

    // 初始化链表 锁
	INIT_LIST_HEAD(&priv->interfaces);
	__mutex_init(&priv->mutex, "subsys mutex", key);
	klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
	klist_init(&priv->klist_drivers, NULL, NULL);

    // 在bus下添加drivers_probe和drivers_autoprobe两个attribute
    // 其中drivers_probe允许用户空间主动发出指定bus下的device_driver的probe动作
    // drivers_autoprobe控制是否在device或device_driver添加到内核时自动执行probe
	retval = add_probe_files(bus);
	if (retval)
		goto bus_probe_files_fail;

	retval = bus_add_groups(bus, bus->bus_groups);
	if (retval)
		goto bus_groups_fail;

	pr_debug("bus: '%s': registered\n", bus->name);
	return 0;

bus_groups_fail:
	remove_probe_files(bus);
bus_probe_files_fail:
	kset_unregister(bus->p->drivers_kset);
bus_drivers_fail:
	kset_unregister(bus->p->devices_kset);
bus_devices_fail:
	bus_remove_file(bus, &bus_attr_uevent);
bus_uevent_fail:
	kset_unregister(&bus->p->subsys);
out:
	kfree(bus->p);
	bus->p = NULL;
	return retval;
}
EXPORT_SYMBOL_GPL(bus_register);
```

device和driver添加到bus中，内核提供了`device_register()`和`driver_register()`两个接口，供各个driver模块使用，这两个接口的核心逻辑是通过bus模块的`bus_add_device()`和`bus_add_driver()`来实现的

`bus_add_device()`的实现
```c
// drivers/base/bus.c
/**
 * bus_add_device - add device to bus
 * @dev: device being added
 *
 * - Add device's bus attributes.
 * - Create links to device's bus.
 * - Add the device to its bus's list of devices.
 */
int bus_add_device(struct device *dev)
{
	struct bus_type *bus = bus_get(dev->bus);
	int error = 0;

	if (bus) {
		pr_debug("bus: '%s': add device %s\n", bus->name, dev_name(dev));
		error = device_add_groups(dev, bus->dev_groups);
		if (error)
			goto out_put;
        // 将该device在sysfs中真正的位置，连接到bus的device目录下
		error = sysfs_create_link(&bus->p->devices_kset->kobj,
						&dev->kobj, dev_name(dev));
		if (error)
			goto out_groups;
        // 为该device的目录下创建一个名为subsystem的链接，链接到该device所在的bus目录
		error = sysfs_create_link(&dev->kobj,
				&dev->bus->p->subsys.kobj, "subsystem");
		if (error)
			goto out_subsys;
        // 将该device添加到链表
		klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devices);
	}
	return 0;

out_subsys:
	sysfs_remove_link(&bus->p->devices_kset->kobj, dev_name(dev));
out_groups:
	device_remove_groups(dev, bus->dev_groups);
out_put:
	bus_put(dev->bus);
	return error;
}

```

#### Class
class是虚拟出来的，为了抽象设备的共性，class为一些相似的device提供通用的接口，定义如下：
```c
struct class {
    const char *name;
    struct module *owner;
    
    const struct attribute_group **class_groups;
    const struct attribute_group **dev_groups;
    struct kobject *dev_kobj;

    int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
    char *(*devnode)(struct device *dev, umode_t *mode);

    void (*class_release)(struct class *class);
    void (*dev_release)(struct device *device);

    int (*shutdown_pre)(struct device *dev);

    const struct kobj_ns_type_operations *ns_type;
    const void *(*namespace)(struct device *dev);

    void (*get_ownership)(struct device *dev, kuid_t *uid, kgid_t *gid);

    const struct dev_pm_ops *pm;

    struct subsys_private *p;
};
```
+ name：class的名称，会在`/sys/class/`下体现
+ class_groups：
+ dev_groups：
+ dev_kobj：表示该class下的设备在`/sys/dev/`下的目录，现在一般有char和block两个，如果dev_kobj为空，则默认选择char
+ class_release：用于release自身的回调函数
+ dev_release：用于release class内设备的回调函数
+ p：class私有数据
