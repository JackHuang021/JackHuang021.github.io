---
title: Linux中断子系统
tags:
  - Linux
  - Interrupt
categories: Linux
abbrlink: 6ea84d7b
date: 2023-08-17 17:46:52
---

## 中断的概念

中断是异步事件的一种形式，CPU需要再执行任务的同时响应外部事件，中断就是一种重要的机制。

在计算机系统中，中断可以分为软件中断和硬件中断两类，在软件中断中，中断是由CPU执行的一条特殊指令造成的，它用于暂停CPU的正常执行流程，然后转而执行一个内部事件或者处理程序，硬件中断则是由硬件设备发出来的，向CPU发出中断请求信号。


## ARMv8 中断

### 异常

ARMv8 指令集架构中，异常分为同步异常和异步异常。

同步异常是试图执行指令时生成的异常，或者是作为指令的执行结果生成的异常，同步异常包括如下：
1. 系统调用
2. 数据终止
3. 指令终止

异步异常包括：
1. 中断
2. 快速中断
3. 系统错误

当异常发生的时候，处理器需要执行异常的处理程序，存储异常处理程序的内存位置称为异常向量，通常把所有异常向量存放在一张表中，称为异常向量表。对于ARMv8处理器的异常级别EL1、EL2、EL3，每个异常级别都有自己的异常向量表，异常向量表的起始虚拟地址存放在寄存器VBAR_ELn（向量基准地址寄存器 Vector Based Address Register）中。

### 中断向量表

每个异常向量表有16项，分为4组，每组4项，每项的长度是128字节（可以存放32条指令）

异常向量表的每一组4个项都是分别表示发生了同步异常，irq，firq和系统错误。这4组的区别就是，第一组表示异常发生在EL0，处理异常的特权等级也是EL0。第二组表示异常发生在ELn（n可以为1，2，3），处理异常的特权等级也是ELn（n可以为1，2，3）

在Linux内核中我们的特权为EL1，我们可以理解为异常发生在EL1，处理异常的特权等级也是EL1。这两组的共同点是异常的发生和处理在同一个特权级别，不需要进行特权级别的切换；而后面两组则是异常的发生和处理不在同一个特权级别，需要进行特权级别的切换。在Linux这里就是说，第三组和第四组表示异常发生在EL0，但是异常处理却在EL1，而他们的区别就是，第三组表示异常发生在64位环境下，第四组表示异常发生在32位环境下。

### 中断亲和性

ARMv8的每一个cpu都有自己的标号，这个标号记录在MPDIR寄存器中，中断的亲和性就是在每一个中断的GICD_IROUTER寄存器中设置亲和性。

## Linux 中断框架

linux 内核中断子系统大致分为 3 部分：
1. 硬件无关的代码，我们称之为内核通用中断处理模块，这些代码是内核中断子系统的核心部分
2. CPU 架构相关的中断处理，和系统使用的具体CPU架构相关
3. 中断控制器驱动代码，和系统使用的中断控制器相关

### 中断号

中断号分为硬件中断号和软件中断号

硬件中断号是中断控制器自己定义的中断号，每一个中断控制器都有0到n不同数量的中断号，用于识别连接到中断控制器的不同的硬件设备发生的中断事件。比如，GIC这个中断控制器，0-15是SGI中断，16-31是PPI中断，32-1023是SPI中断，我们的设备到是连接到SPI中断上的，甚至GPIO中断控制器也是连接到SPI上面的。GPIO中断控制器自己也有这0-n的中断。所以硬件中断号是某一个中断控制器上的硬件标识。

软件中断号，有时候也叫虚拟中断号，是由内核定义的中断号，用于执行内核函数或服务。软件中断号是预定的，由系统中的中断控制器管理。硬件中断号的产生都会分配一个对应的软件中断号。所以软件中断号硬件无关，仅仅是被CPU用来标识一个外设中断，便于系统管理所有的中断。

对于驱动工程师而言，我们和CPU视角是一样的，我们只希望得到一个软件中断号，而不关系具体是那个interrupt controller上的那个硬件中断号。这样一个好处是在中断相关的硬件发生变化的时候，驱动软件不需要修改。因此，linux内核中的中断子系统需要提供一个将硬件中断号映射到软件中断号上来的机制。

## GIC V3 中断控制器

ARM多核处理器里最常用的中断控制器是GIC， GIC是Generic Interrupt Controller的缩写，提供了灵活的和可扩展的中断管理方法，支持单核系统到数百个大型多芯片设计的核心。 主要作用就是接受硬件中断信号，通过一定的设置策略，然后分发给对应的CPU进行处理。

GIC是由ARM公司提出设计规范，当前有四个版本，GIC v1~v4。设计规范中最常用的，有3个版本V2.0、V3.1、V4.1，GICv3版本设计主要运行在Armv8-A, Armv9-A等架构上。 ARM公司并给出一个实际的控制器设计参考，比如GIC-400(支持GIC v2架构)、gic500(支持GIC v3架构)、GIC-600(支持GIC v3和GIC v4架构)。 最终芯片厂商可以自己实现GIC或者直接购买ARM提供的设计。

### 中断子系统的一些概念

+ HW Interrut ID： 硬件中断ID，
+ IRQ Number：虚拟终端ID
+ IRQ Domain：
+ 中断向量表：中断向量表在内存中存放，表中存放着中断源所对应的中断处理程序的入口地址

### GIC V3 中断类型

+ SGI（Software Generated Interrupt）: 软件触发的中断，软件可以通过写 GICD_SGIR 寄存器来触发一个中断事件，一般用于核间通信，内核中的 IPI 中断就是基于SGI。
+ PPI（Private Peripheral Interrupt）：私有外设中断，该中断来自于外设，被特定的核处理，GIC 是支持多核的，每个核有自己独立的中断。
+ SPI（Shared Peripheral Interrupt）：共享外设中断，所有核共享的中断，中断产生后，可以分发到某一个 CPU 上
+ LPI）（Locality-specific Peripheral Interrupt）：LPI 是在 GIC V3 中引入的，并且与其他三种类型的中断具有非常不同的编程模型，LPI 是基于消息的中断，它们的配置保存在表中而不是寄存器。

每一个中断都有一个 ID 号标识，称为 INTID，下面是 ARM GIC V3 手册的中断号规定：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250605140902.png)

### 中断状态和处理流程

一个简单的中断处理过程是：外设发起中断，发送给 Distributor，Distributor 并基于它们的中断特性（优先级、是否使能等等）对中断进行分发处理，分发给合适的 Redistributor，Redistributor 将中断信息发送给 CPU Interface，CPU Interface 产生合适的中断异常给处理器，处理器接收该异常，最后软件处理该中断。

### GIC V3 寄存器

GIC V3 寄存器的分布图：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250605142304.png)

GICC 开头的表示 CPU Interface 寄存器，GICD 表示 Distributor 寄存器，GITS 表示 ITS 寄存器，GICR 表示 Redistributor 寄存器，CPU Interface 寄存器提供了 2 种访问方式，一种是 Memory-mapped 的访问，一种是系统寄存器的访问，

## GIC V3 中断驱动分析

### E2000 GIC 设备树描述

```c
gic: interrupt-controller@30800000 {
	compatible = "arm,gic-v3";
	#interrupt-cells = <3>;
	#address-cells = <2>;
	#size-cells = <2>;
	ranges;
	interrupt-controller;
	reg = <0x0 0x30800000 0 0x20000>,	/* GICD */
			<0x0 0x30880000 0 0x80000>,	/* GICR */
			<0x0 0x30840000 0 0x10000>,	/* GICC */
			<0x0 0x30850000 0 0x10000>,	/* GICH */
			<0x0 0x30860000 0 0x10000>;	/* GICV */
	interrupts = <GIC_PPI 9 IRQ_TYPE_LEVEL_LOW>;

	its: gic-its@30820000 {
		compatible = "arm,gic-v3-its";
		msi-controller;
		reg = <0x0 0x30820000 0x0 0x20000>;
	};
};
```

+ interrupt-controller： 声明该设备树节点是一个中断控制器
+ #interrupt-cells ：指定使用该中断控制器的节点要用几个cells来描述一个中断，可理解为用几个参数来描述一个中断信息。 在这里的意思是在intc节点的子节点将用3个参数来描述中断。
+ its： 在gic设备节点下，有一个子设备节点its，ITS设备用于将消息信号中断(MSI)路由到cpu
+ msi-controller ： 标识该设备是MSI控制器

### 驱动代码分析

`IRQCHIP_DECLARE`宏初始化了一个`struct of_device_id`静态常量，放到并放置在`__irqchip_of_table`段中。

```c
// drivers/irqchip/irq-gic-v3.c 
IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);

// include/linux/irqchip.h
/*
 * This macro must be used by the different irqchip drivers to declare
 * the association between their DT compatible string and their
 * initialization function.
 *
 * @name: name that must be unique across all IRQCHIP_DECLARE of the
 * same file.
 * @compat: compatible string of the irqchip driver
 * @fn: initialization function
 */
#define IRQCHIP_DECLARE(name, compat, fn)	\
	OF_DECLARE_2(irqchip, name, compat, typecheck_irq_init_cb(fn))

// include/linux/of.h
#define OF_DECLARE_2(table, name, compat, fn) \
	_OF_DECLARE(table, name, compat, fn, of_init_fn_2)

#define _OF_DECLARE(table, name, compat, fn, fn_type)			\
static const struct of_device_id __of_table_##name		\
	__used __section("__" #table "_of_table")		\
	__aligned(__alignof__(struct of_device_id))		\
		= { .compatible = compat,				\
			.data = (fn == (fn_type)NULL) ? fn : fn  }
```

### 相关结构体

#### `struct gic_chip_data`

```c
// drivers/irqchip/irq-gic-v3.c
struct gic_chip_data {
	// 指向 GIC 设备节点
	struct fwnode_handle	*fwnode;
	// 保存 GICD 寄存器的物理基地址
	phys_addr_t		dist_phys_base;
	// 保存 GICD 寄存器映射后的基地址
	void __iomem		*dist_base;
	// 保存 GICR 寄存器的地址信息
	struct redist_region	*redist_regions;
	struct rdists		rdists;
	// 指向 irq_domain
	struct irq_domain	*domain;
	u64			redist_stride;
	u32			nr_redist_regions;
	u64			flags;
	bool			has_rss;
	unsigned int		ppi_nr;
	struct partition_desc	**ppi_descs;
};
```

#### `struct irq_domain`

```c
// include/linux/irqdomain.h
/**
 * struct irq_domain - Hardware interrupt number translation object
 * @link:	Element in global irq_domain list.
 * @name:	Name of interrupt domain
 * @ops:	Pointer to irq_domain methods
 * @host_data:	Private data pointer for use by owner.  Not touched by irq_domain
 *		core code.
 * @flags:	Per irq_domain flags
 * @mapcount:	The number of mapped interrupts
 * @mutex:	Domain lock, hierarchical domains use root domain's lock
 * @root:	Pointer to root domain, or containing structure if non-hierarchical
 *
 * Optional elements:
 * @fwnode:	Pointer to firmware node associated with the irq_domain. Pretty easy
 *		to swap it for the of_node via the irq_domain_get_of_node accessor
 * @gc:		Pointer to a list of generic chips. There is a helper function for
 *		setting up one or more generic chips for interrupt controllers
 *		drivers using the generic chip library which uses this pointer.
 * @dev:	Pointer to the device which instantiated the irqdomain
 *		With per device irq domains this is not necessarily the same
 *		as @pm_dev.
 * @pm_dev:	Pointer to a device that can be utilized for power management
 *		purposes related to the irq domain.
 * @parent:	Pointer to parent irq_domain to support hierarchy irq_domains
 * @msi_parent_ops: Pointer to MSI parent domain methods for per device domain init
 *
 * Revmap data, used internally by the irq domain code:
 * @revmap_size:	Size of the linear map table @revmap[]
 * @revmap_tree:	Radix map tree for hwirqs that don't fit in the linear map
 * @revmap:		Linear table of irq_data pointers
 */
struct irq_domain {
	struct list_head		link;
	const char			*name;
	const struct irq_domain_ops	*ops;
	void				*host_data; 
	// 存储irq domain的一些标志
	unsigned int			flags;
	unsigned int			mapcount;
	struct mutex			mutex;
	struct irq_domain		*root;

	/* Optional data */
	struct fwnode_handle		*fwnode;
	enum irq_domain_bus_token	bus_token;
	struct irq_domain_chip_generic	*gc;
	struct device			*dev;
	struct device			*pm_dev;
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	struct irq_domain		*parent;
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
	const struct msi_parent_ops	*msi_parent_ops;
#endif

	/* reverse map data. The linear map gets appended to the irq_domain */
	irq_hw_number_t			hwirq_max;
	unsigned int			revmap_size;
	struct radix_tree_root		revmap_tree;
	struct irq_data __rcu		*revmap[];
};

/* Irq domain flags */
enum {
	/* Irq domain is hierarchical */
	IRQ_DOMAIN_FLAG_HIERARCHY	= (1 << 0),

	/* Irq domain name was allocated in __irq_domain_add() */
	IRQ_DOMAIN_NAME_ALLOCATED	= (1 << 1),

	/* Irq domain is an IPI domain with virq per cpu */
	IRQ_DOMAIN_FLAG_IPI_PER_CPU	= (1 << 2),

	/* Irq domain is an IPI domain with single virq */
	IRQ_DOMAIN_FLAG_IPI_SINGLE	= (1 << 3),

	/* Irq domain implements MSIs */
	IRQ_DOMAIN_FLAG_MSI		= (1 << 4),

	/*
	 * Irq domain implements isolated MSI, see msi_device_has_isolated_msi()
	 */
	IRQ_DOMAIN_FLAG_ISOLATED_MSI	= (1 << 5),

	/* Irq domain doesn't translate anything */
	IRQ_DOMAIN_FLAG_NO_MAP		= (1 << 6),

	/* Irq domain is a MSI parent domain */
	IRQ_DOMAIN_FLAG_MSI_PARENT	= (1 << 8),

	/* Irq domain is a MSI device domain */
	IRQ_DOMAIN_FLAG_MSI_DEVICE	= (1 << 9),

	/*
	 * Flags starting from IRQ_DOMAIN_FLAG_NONCORE are reserved
	 * for implementation specific purposes and ignored by the
	 * core code.
	 */
	IRQ_DOMAIN_FLAG_NONCORE		= (1 << 16),
};
```


### 数据结构描述
```c
// include/linux/irqdomain.h
/**
 * struct irq_domain - Hardware interrupt number translation object
 * @link: Element in global irq_domain list.
 * @name: Name of interrupt domain
 * @ops: pointer to irq_domain methods
 * @host_data: private data pointer for use by owner.  Not touched by irq_domain
 *             core code.
 * @flags: host per irq_domain flags
 * @mapcount: The number of mapped interrupts
 *
 * Optional elements
 * @fwnode: Pointer to firmware node associated with the irq_domain. Pretty easy
 *          to swap it for the of_node via the irq_domain_get_of_node accessor
 * @gc: Pointer to a list of generic chips. There is a helper function for
 *      setting up one or more generic chips for interrupt controllers
 *      drivers using the generic chip library which uses this pointer.
 * @parent: Pointer to parent irq_domain to support hierarchy irq_domains
 * @debugfs_file: dentry for the domain debugfs file
 *
 * Revmap data, used internally by irq_domain
 * @revmap_direct_max_irq: The largest hwirq that can be set for controllers that
 *                         support direct mapping
 * @revmap_size: Size of the linear map table @linear_revmap[]
 * @revmap_tree: Radix map tree for hwirqs that don't fit in the linear map
 * @linear_revmap: Linear table of hwirq->virq reverse mappings
 */
struct irq_domain {
    // 所有irq domain会链接到内核irq_domain_list链表中
	struct list_head link;
	const char *name;
    // irq domain的回调函数
	const struct irq_domain_ops *ops;
	// host_data定义了中断控制器使用的私有数据
	void *host_data;
	unsigned int flags;
	unsigned int mapcount;

	/* Optional data */
    // irq domain对应的中断控制器的device node
	struct fwnode_handle *fwnode;
	enum irq_domain_bus_token bus_token;
	struct irq_domain_chip_generic *gc;
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	struct irq_domain *parent;
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
	struct dentry		*debugfs_file;
#endif

	/* reverse map data. The linear map gets appended to the irq_domain */
    // domain中最大的hw irq ID
	irq_hw_number_t hwirq_max;
	unsigned int revmap_direct_max_irq;
    // 线性映射表的大小
	unsigned int revmap_size;
	struct radix_tree_root revmap_tree;
	struct mutex revmap_tree_mutex;
	// 线性表使用的查找表
	unsigned int linear_revmap[];
};
```


```c
// include/linux/irqdomain.h
/**
 * struct irq_domain_ops - Methods for irq_domain objects
 * @match: Match an interrupt controller device node to a host, returns
 *         1 on a match
 * @map: Create or update a mapping between a virtual irq number and a hw
 *       irq number. This is called only once for a given mapping.
 * @unmap: Dispose of such a mapping
 * @xlate: Given a device tree node and interrupt specifier, decode
 *         the hardware irq number and linux irq type value.
 *
 * Functions below are provided by the driver and called whenever a new mapping
 * is created or an old mapping is disposed. The driver can then proceed to
 * whatever internal data structures management is required. It also needs
 * to setup the irq_desc when returning from map().
 */
struct irq_domain_ops {
	int (*match)(struct irq_domain *d, struct device_node *node,
		     enum irq_domain_bus_token bus_token);
	int (*select)(struct irq_domain *d, struct irq_fwspec *fwspec,
		      enum irq_domain_bus_token bus_token);
	int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
	void (*unmap)(struct irq_domain *d, unsigned int virq);
	int (*xlate)(struct irq_domain *d, struct device_node *node,
		     const u32 *intspec, unsigned int intsize,
		     unsigned long *out_hwirq, unsigned int *out_type);
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	/* extended V2 interfaces to support hierarchy irq_domains */
	int (*alloc)(struct irq_domain *d, unsigned int virq,
		     unsigned int nr_irqs, void *arg);
	void (*free)(struct irq_domain *d, unsigned int virq,
		     unsigned int nr_irqs);
	int (*activate)(struct irq_domain *d, struct irq_data *irqd, bool reserve);
	void (*deactivate)(struct irq_domain *d, struct irq_data *irq_data);
	int (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
			 unsigned long *out_hwirq, unsigned int *out_type);
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
	void (*debug_show)(struct seq_file *m, struct irq_domain *d,
			   struct irq_data *irqd, int ind);
#endif
};
```

### 设备树中中断控制器的描述
```c
// 设备树根节点interrupt-parent描述
/ {
	compatible = "phytium,pe220x";
	// 表明外设的interrupt request lint物理连接到了gic中断控制器
	// 在device node中没有定义interrupt-parent属性的均使用这个中断控制器
	interrupt-parent = <&gic>;

// pe220x系列gic中断控制器设备树节点
gic: interrupt-controller@30800000 {
	compatible = "arm,gic-v3";
	// 该中断控制器用多少个cell描述一个外设的interrupt request line
	#interrupt-cells = <3>;
	#address-cells = <2>;
	#size-cells = <2>;
	ranges;
	// 表明该device node是一个中断控制器
	interrupt-controller;
	reg = <0x0 0x30800000 0 0x20000>,	/* GICD */
			<0x0 0x30880000 0 0x80000>,	/* GICR */
			<0x0 0x30840000 0 0x10000>,	/* GICC */
			<0x0 0x30850000 0 0x10000>,	/* GICH */
			<0x0 0x30860000 0 0x10000>;	/* GICV */
	interrupts = <GIC_PPI 9 IRQ_TYPE_LEVEL_LOW>;

	its: gic-its@30820000 {
		compatible = "arm,gic-v3-its";
		msi-controller;
		reg = <0x0 0x30820000 0x0 0x20000>;
	};
};
```

### 中断映射的建立
中断映射建立过程如下：
1. dts文件描述了系统中interrupt controller以及外设IRQ的拓扑结构
2. device tree初始化的时候，形成了系统内所有device node的树状结构
3. 在初始化的时候会调用`of_irq_init()`扫描所有interrupt controller节点，并调用合适的interrupt controller driver进行初始化，并在driver的初始化过程中创建映射


## GPIO 中断分析

### E2000 GPIO 设备树描述

```c
gpio0: gpio@28034000 {
	compatible = "phytium,gpio";
	reg = <0x0 0x28034000 0x0 0x1000>;
	interrupts = <GIC_SPI 108 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 109 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 110 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 111 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 112 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 113 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 114 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 115 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 116 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 117 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 118 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 119 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 120 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 121 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 122 IRQ_TYPE_LEVEL_HIGH>,
				<GIC_SPI 123 IRQ_TYPE_LEVEL_HIGH>;
	gpio-controller;
	#gpio-cells = <2>;
	#address-cells = <1>;
	#size-cells = <0>;
	status = "disabled";

	porta {
		compatible = "phytium,gpio-port";
		reg = <0>;
		ngpios = <16>;
	};
};
```

### 相关结构体

#### `struct gpio_chip`

`struct gpio_chip` 用于抽象一个GPIO控制器

```c
/**
 * struct gpio_chip - abstract a GPIO controller
 * @label: a functional name for the GPIO device, such as a part
 *	number or the name of the SoC IP-block implementing it.
 * @gpiodev: the internal state holder, opaque struct
 * @parent: optional parent device providing the GPIOs
 * @fwnode: optional fwnode providing this controller's properties
 * @owner: helps prevent removal of modules exporting active GPIOs
 * @request: optional hook for chip-specific activation, such as
 *	enabling module power and clock; may sleep
 * @free: optional hook for chip-specific deactivation, such as
 *	disabling module power and clock; may sleep
 * @get_direction: returns direction for signal "offset", 0=out, 1=in,
 *	(same as GPIO_LINE_DIRECTION_OUT / GPIO_LINE_DIRECTION_IN),
 *	or negative error. It is recommended to always implement this
 *	function, even on input-only or output-only gpio chips.
 * @direction_input: configures signal "offset" as input, or returns error
 *	This can be omitted on input-only or output-only gpio chips.
 * @direction_output: configures signal "offset" as output, or returns error
 *	This can be omitted on input-only or output-only gpio chips.
 * @get: returns value for signal "offset", 0=low, 1=high, or negative error
 * @get_multiple: reads values for multiple signals defined by "mask" and
 *	stores them in "bits", returns 0 on success or negative error
 * @set: assigns output value for signal "offset"
 * @set_multiple: assigns output values for multiple signals defined by "mask"
 * @set_config: optional hook for all kinds of settings. Uses the same
 *	packed config format as generic pinconf.
 * @to_irq: optional hook supporting non-static gpiod_to_irq() mappings;
 *	implementation may not sleep
 * @dbg_show: optional routine to show contents in debugfs; default code
 *	will be used when this is omitted, but custom code can show extra
 *	state (such as pullup/pulldown configuration).
 * @init_valid_mask: optional routine to initialize @valid_mask, to be used if
 *	not all GPIOs are valid.
 * @add_pin_ranges: optional routine to initialize pin ranges, to be used when
 *	requires special mapping of the pins that provides GPIO functionality.
 *	It is called after adding GPIO chip and before adding IRQ chip.
 * @en_hw_timestamp: Dependent on GPIO chip, an optional routine to
 *	enable hardware timestamp.
 * @dis_hw_timestamp: Dependent on GPIO chip, an optional routine to
 *	disable hardware timestamp.
 * @base: identifies the first GPIO number handled by this chip;
 *	or, if negative during registration, requests dynamic ID allocation.
 *	DEPRECATION: providing anything non-negative and nailing the base
 *	offset of GPIO chips is deprecated. Please pass -1 as base to
 *	let gpiolib select the chip base in all possible cases. We want to
 *	get rid of the static GPIO number space in the long run.
 * @ngpio: the number of GPIOs handled by this controller; the last GPIO
 *	handled is (base + ngpio - 1).
 * @offset: when multiple gpio chips belong to the same device this
 *	can be used as offset within the device so friendly names can
 *	be properly assigned.
 * @names: if set, must be an array of strings to use as alternative
 *      names for the GPIOs in this chip. Any entry in the array
 *      may be NULL if there is no alias for the GPIO, however the
 *      array must be @ngpio entries long.  A name can include a single printk
 *      format specifier for an unsigned int.  It is substituted by the actual
 *      number of the gpio.
 * @can_sleep: flag must be set iff get()/set() methods sleep, as they
 *	must while accessing GPIO expander chips over I2C or SPI. This
 *	implies that if the chip supports IRQs, these IRQs need to be threaded
 *	as the chip access may sleep when e.g. reading out the IRQ status
 *	registers.
 * @read_reg: reader function for generic GPIO
 * @write_reg: writer function for generic GPIO
 * @be_bits: if the generic GPIO has big endian bit order (bit 31 is representing
 *	line 0, bit 30 is line 1 ... bit 0 is line 31) this is set to true by the
 *	generic GPIO core. It is for internal housekeeping only.
 * @reg_dat: data (in) register for generic GPIO
 * @reg_set: output set register (out=high) for generic GPIO
 * @reg_clr: output clear register (out=low) for generic GPIO
 * @reg_dir_out: direction out setting register for generic GPIO
 * @reg_dir_in: direction in setting register for generic GPIO
 * @bgpio_dir_unreadable: indicates that the direction register(s) cannot
 *	be read and we need to rely on out internal state tracking.
 * @bgpio_bits: number of register bits used for a generic GPIO i.e.
 *	<register width> * 8
 * @bgpio_lock: used to lock chip->bgpio_data. Also, this is needed to keep
 *	shadowed and real data registers writes together.
 * @bgpio_data:	shadowed data register for generic GPIO to clear/set bits
 *	safely.
 * @bgpio_dir: shadowed direction register for generic GPIO to clear/set
 *	direction safely. A "1" in this word means the line is set as
 *	output.
 *
 * A gpio_chip can help platforms abstract various sources of GPIOs so
 * they can all be accessed through a common programming interface.
 * Example sources would be SOC controllers, FPGAs, multifunction
 * chips, dedicated GPIO expanders, and so on.
 *
 * Each chip controls a number of signals, identified in method calls
 * by "offset" values in the range 0..(@ngpio - 1).  When those signals
 * are referenced through calls like gpio_get_value(gpio), the offset
 * is calculated by subtracting @base from the gpio number.
 */
struct gpio_chip {
	const char		*label;
	// 指向一个 gpio_device 结构体
	struct gpio_device	*gpiodev;
	struct device		*parent;
	struct fwnode_handle	*fwnode;
	struct module		*owner;

	int			(*request)(struct gpio_chip *gc,
						unsigned int offset);
	void			(*free)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*get_direction)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*direction_input)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*direction_output)(struct gpio_chip *gc,
						unsigned int offset, int value);
	int			(*get)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*get_multiple)(struct gpio_chip *gc,
						unsigned long *mask,
						unsigned long *bits);
	void			(*set)(struct gpio_chip *gc,
						unsigned int offset, int value);
	void			(*set_multiple)(struct gpio_chip *gc,
						unsigned long *mask,
						unsigned long *bits);
	int			(*set_config)(struct gpio_chip *gc,
					      unsigned int offset,
					      unsigned long config);
	int			(*to_irq)(struct gpio_chip *gc,
						unsigned int offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *gc);

	int			(*init_valid_mask)(struct gpio_chip *gc,
						   unsigned long *valid_mask,
						   unsigned int ngpios);

	int			(*add_pin_ranges)(struct gpio_chip *gc);

	int			(*en_hw_timestamp)(struct gpio_chip *gc,
						   u32 offset,
						   unsigned long flags);
	int			(*dis_hw_timestamp)(struct gpio_chip *gc,
						    u32 offset,
						    unsigned long flags);
	int			base;
	u16			ngpio;
	u16			offset;
	const char		*const *names;
	bool			can_sleep;

#if IS_ENABLED(CONFIG_GPIO_GENERIC)
	unsigned long (*read_reg)(void __iomem *reg);
	void (*write_reg)(void __iomem *reg, unsigned long data);
	bool be_bits;
	void __iomem *reg_dat;
	void __iomem *reg_set;
	void __iomem *reg_clr;
	void __iomem *reg_dir_out;
	void __iomem *reg_dir_in;
	bool bgpio_dir_unreadable;
	int bgpio_bits;
	raw_spinlock_t bgpio_lock;
	unsigned long bgpio_data;
	unsigned long bgpio_dir;
#endif /* CONFIG_GPIO_GENERIC */

#ifdef CONFIG_GPIOLIB_IRQCHIP
	/*
	 * With CONFIG_GPIOLIB_IRQCHIP we get an irqchip inside the gpiolib
	 * to handle IRQs for most practical cases.
	 */

	/**
	 * @irq:
	 *
	 * Integrates interrupt chip functionality with the GPIO chip. Can be
	 * used to handle IRQs for most practical cases.
	 */
	struct gpio_irq_chip irq;
#endif /* CONFIG_GPIOLIB_IRQCHIP */

	/**
	 * @valid_mask:
	 *
	 * If not %NULL, holds bitmask of GPIOs which are valid to be used
	 * from the chip.
	 */
	unsigned long *valid_mask;

#if defined(CONFIG_OF_GPIO)
	/*
	 * If CONFIG_OF_GPIO is enabled, then all GPIO controllers described in
	 * the device tree automatically may have an OF translation
	 */

	/**
	 * @of_gpio_n_cells:
	 *
	 * Number of cells used to form the GPIO specifier.
	 */
	unsigned int of_gpio_n_cells;

	/**
	 * @of_xlate:
	 *
	 * Callback to translate a device tree GPIO specifier into a chip-
	 * relative GPIO number and flags.
	 */
	int (*of_xlate)(struct gpio_chip *gc,
			const struct of_phandle_args *gpiospec, u32 *flags);
#endif /* CONFIG_OF_GPIO */
};
```

#### `struct gpio_irq_chip`

```c
// include/linux/gpio/driver.h
/**
 * struct gpio_irq_chip - GPIO interrupt controller
 */
struct gpio_irq_chip {
	/**
	 * @chip:
	 *
	 * GPIO IRQ chip implementation, provided by GPIO driver.
	 */
	struct irq_chip *chip;

	/**
	 * @domain:
	 *
	 * Interrupt translation domain; responsible for mapping between GPIO
	 * hwirq number and Linux IRQ number.
	 */
	struct irq_domain *domain;

#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
	/**
	 * @fwnode:
	 *
	 * Firmware node corresponding to this gpiochip/irqchip, necessary
	 * for hierarchical irqdomain support.
	 */
	struct fwnode_handle *fwnode;

	/**
	 * @parent_domain:
	 *
	 * If non-NULL, will be set as the parent of this GPIO interrupt
	 * controller's IRQ domain to establish a hierarchical interrupt
	 * domain. The presence of this will activate the hierarchical
	 * interrupt support.
	 */
	struct irq_domain *parent_domain;

	/**
	 * @child_to_parent_hwirq:
	 *
	 * This callback translates a child hardware IRQ offset to a parent
	 * hardware IRQ offset on a hierarchical interrupt chip. The child
	 * hardware IRQs correspond to the GPIO index 0..ngpio-1 (see the
	 * ngpio field of struct gpio_chip) and the corresponding parent
	 * hardware IRQ and type (such as IRQ_TYPE_*) shall be returned by
	 * the driver. The driver can calculate this from an offset or using
	 * a lookup table or whatever method is best for this chip. Return
	 * 0 on successful translation in the driver.
	 *
	 * If some ranges of hardware IRQs do not have a corresponding parent
	 * HWIRQ, return -EINVAL, but also make sure to fill in @valid_mask and
	 * @need_valid_mask to make these GPIO lines unavailable for
	 * translation.
	 */
	int (*child_to_parent_hwirq)(struct gpio_chip *gc,
				     unsigned int child_hwirq,
				     unsigned int child_type,
				     unsigned int *parent_hwirq,
				     unsigned int *parent_type);

	/**
	 * @populate_parent_alloc_arg :
	 *
	 * This optional callback allocates and populates the specific struct
	 * for the parent's IRQ domain. If this is not specified, then
	 * &gpiochip_populate_parent_fwspec_twocell will be used. A four-cell
	 * variant named &gpiochip_populate_parent_fwspec_fourcell is also
	 * available.
	 */
	int (*populate_parent_alloc_arg)(struct gpio_chip *gc,
					 union gpio_irq_fwspec *fwspec,
					 unsigned int parent_hwirq,
					 unsigned int parent_type);

	/**
	 * @child_offset_to_irq:
	 *
	 * This optional callback is used to translate the child's GPIO line
	 * offset on the GPIO chip to an IRQ number for the GPIO to_irq()
	 * callback. If this is not specified, then a default callback will be
	 * provided that returns the line offset.
	 */
	unsigned int (*child_offset_to_irq)(struct gpio_chip *gc,
					    unsigned int pin);

	/**
	 * @child_irq_domain_ops:
	 *
	 * The IRQ domain operations that will be used for this GPIO IRQ
	 * chip. If no operations are provided, then default callbacks will
	 * be populated to setup the IRQ hierarchy. Some drivers need to
	 * supply their own translate function.
	 */
	struct irq_domain_ops child_irq_domain_ops;
#endif

	/**
	 * @handler:
	 *
	 * The IRQ handler to use (often a predefined IRQ core function) for
	 * GPIO IRQs, provided by GPIO driver.
	 */
	irq_flow_handler_t handler;

	/**
	 * @default_type:
	 *
	 * Default IRQ triggering type applied during GPIO driver
	 * initialization, provided by GPIO driver.
	 */
	unsigned int default_type;

	/**
	 * @lock_key:
	 *
	 * Per GPIO IRQ chip lockdep class for IRQ lock.
	 */
	struct lock_class_key *lock_key;

	/**
	 * @request_key:
	 *
	 * Per GPIO IRQ chip lockdep class for IRQ request.
	 */
	struct lock_class_key *request_key;

	/**
	 * @parent_handler:
	 *
	 * The interrupt handler for the GPIO chip's parent interrupts, may be
	 * NULL if the parent interrupts are nested rather than cascaded.
	 */
	/* 指向中断处理函数 */
	irq_flow_handler_t parent_handler;

	union {
		/**
		 * @parent_handler_data:
		 *
		 * If @per_parent_data is false, @parent_handler_data is a
		 * single pointer used as the data associated with every
		 * parent interrupt.
		 */
		void *parent_handler_data;

		/**
		 * @parent_handler_data_array:
		 *
		 * If @per_parent_data is true, @parent_handler_data_array is
		 * an array of @num_parents pointers, and is used to associate
		 * different data for each parent. This cannot be NULL if
		 * @per_parent_data is true.
		 */
		void **parent_handler_data_array;
	};

	/**
	 * @num_parents:
	 *
	 * The number of interrupt parents of a GPIO chip.
	 */
	/* GPIO 控制器有多少个硬件中断 */
	unsigned int num_parents;

	/**
	 * @parents:
	 *
	 * A list of interrupt parents of a GPIO chip. This is owned by the
	 * driver, so the core will only reference this list, not modify it.
	 */
	/* 指向GPIO的硬件中断号 */
	unsigned int *parents;

	/**
	 * @map:
	 *
	 * A list of interrupt parents for each line of a GPIO chip.
	 */
	unsigned int *map;

	/**
	 * @threaded:
	 *
	 * True if set the interrupt handling uses nested threads.
	 */
	bool threaded;

	/**
	 * @per_parent_data:
	 *
	 * True if parent_handler_data_array describes a @num_parents
	 * sized array to be used as parent data.
	 */
	bool per_parent_data;

	/**
	 * @initialized:
	 *
	 * Flag to track GPIO chip irq member's initialization.
	 * This flag will make sure GPIO chip irq members are not used
	 * before they are initialized.
	 */
	bool initialized;

	/**
	 * @domain_is_allocated_externally:
	 *
	 * True it the irq_domain was allocated outside of gpiolib, in which
	 * case gpiolib won't free the irq_domain itself.
	 */
	bool domain_is_allocated_externally;

	/**
	 * @init_hw: optional routine to initialize hardware before
	 * an IRQ chip will be added. This is quite useful when
	 * a particular driver wants to clear IRQ related registers
	 * in order to avoid undesired events.
	 */
	int (*init_hw)(struct gpio_chip *gc);

	/**
	 * @init_valid_mask: optional routine to initialize @valid_mask, to be
	 * used if not all GPIO lines are valid interrupts. Sometimes some
	 * lines just cannot fire interrupts, and this routine, when defined,
	 * is passed a bitmap in "valid_mask" and it will have ngpios
	 * bits from 0..(ngpios-1) set to "1" as in valid. The callback can
	 * then directly set some bits to "0" if they cannot be used for
	 * interrupts.
	 */
	void (*init_valid_mask)(struct gpio_chip *gc,
				unsigned long *valid_mask,
				unsigned int ngpios);

	/**
	 * @valid_mask:
	 *
	 * If not %NULL, holds bitmask of GPIOs which are valid to be included
	 * in IRQ domain of the chip.
	 */
	unsigned long *valid_mask;

	/**
	 * @first:
	 *
	 * Required for static IRQ allocation. If set, irq_domain_add_simple()
	 * will allocate and map all IRQs during initialization.
	 */
	unsigned int first;

	/**
	 * @irq_enable:
	 *
	 * Store old irq_chip irq_enable callback
	 */
	void		(*irq_enable)(struct irq_data *data);

	/**
	 * @irq_disable:
	 *
	 * Store old irq_chip irq_disable callback
	 */
	void		(*irq_disable)(struct irq_data *data);
	/**
	 * @irq_unmask:
	 *
	 * Store old irq_chip irq_unmask callback
	 */
	void		(*irq_unmask)(struct irq_data *data);

	/**
	 * @irq_mask:
	 *
	 * Store old irq_chip irq_mask callback
	 */
	void		(*irq_mask)(struct irq_data *data);
};
```

#### `struct irq_chip`

```c
// include/linux/irq.h
/**
 * struct irq_chip - hardware interrupt chip descriptor
 *
 * @name:		name for /proc/interrupts
 * @irq_startup:	start up the interrupt (defaults to ->enable if NULL)
 * @irq_shutdown:	shut down the interrupt (defaults to ->disable if NULL)
 * @irq_enable:		enable the interrupt (defaults to chip->unmask if NULL)
 * @irq_disable:	disable the interrupt
 * @irq_ack:		start of a new interrupt
 * @irq_mask:		mask an interrupt source
 * @irq_mask_ack:	ack and mask an interrupt source
 * @irq_unmask:		unmask an interrupt source
 * @irq_eoi:		end of interrupt
 * @irq_set_affinity:	Set the CPU affinity on SMP machines. If the force
 *			argument is true, it tells the driver to
 *			unconditionally apply the affinity setting. Sanity
 *			checks against the supplied affinity mask are not
 *			required. This is used for CPU hotplug where the
 *			target CPU is not yet set in the cpu_online_mask.
 * @irq_retrigger:	resend an IRQ to the CPU
 * @irq_set_type:	set the flow type (IRQ_TYPE_LEVEL/etc.) of an IRQ
 * @irq_set_wake:	enable/disable power-management wake-on of an IRQ
 * @irq_bus_lock:	function to lock access to slow bus (i2c) chips
 * @irq_bus_sync_unlock:function to sync and unlock slow bus (i2c) chips
 * @irq_cpu_online:	configure an interrupt source for a secondary CPU
 * @irq_cpu_offline:	un-configure an interrupt source for a secondary CPU
 * @irq_suspend:	function called from core code on suspend once per
 *			chip, when one or more interrupts are installed
 * @irq_resume:		function called from core code on resume once per chip,
 *			when one ore more interrupts are installed
 * @irq_pm_shutdown:	function called from core code on shutdown once per chip
 * @irq_calc_mask:	Optional function to set irq_data.mask for special cases
 * @irq_print_chip:	optional to print special chip info in show_interrupts
 * @irq_request_resources:	optional to request resources before calling
 *				any other callback related to this irq
 * @irq_release_resources:	optional to release resources acquired with
 *				irq_request_resources
 * @irq_compose_msi_msg:	optional to compose message content for MSI
 * @irq_write_msi_msg:	optional to write message content for MSI
 * @irq_get_irqchip_state:	return the internal state of an interrupt
 * @irq_set_irqchip_state:	set the internal state of a interrupt
 * @irq_set_vcpu_affinity:	optional to target a vCPU in a virtual machine
 * @ipi_send_single:	send a single IPI to destination cpus
 * @ipi_send_mask:	send an IPI to destination cpus in cpumask
 * @irq_nmi_setup:	function called from core code before enabling an NMI
 * @irq_nmi_teardown:	function called from core code after disabling an NMI
 * @flags:		chip specific flags
 */
struct irq_chip {
	const char	*name;
	unsigned int	(*irq_startup)(struct irq_data *data);
	void		(*irq_shutdown)(struct irq_data *data);
	void		(*irq_enable)(struct irq_data *data);
	void		(*irq_disable)(struct irq_data *data);

	void		(*irq_ack)(struct irq_data *data);
	void		(*irq_mask)(struct irq_data *data);
	void		(*irq_mask_ack)(struct irq_data *data);
	void		(*irq_unmask)(struct irq_data *data);
	void		(*irq_eoi)(struct irq_data *data);

	int		(*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force);
	int		(*irq_retrigger)(struct irq_data *data);
	int		(*irq_set_type)(struct irq_data *data, unsigned int flow_type);
	int		(*irq_set_wake)(struct irq_data *data, unsigned int on);

	void		(*irq_bus_lock)(struct irq_data *data);
	void		(*irq_bus_sync_unlock)(struct irq_data *data);

#ifdef CONFIG_DEPRECATED_IRQ_CPU_ONOFFLINE
	void		(*irq_cpu_online)(struct irq_data *data);
	void		(*irq_cpu_offline)(struct irq_data *data);
#endif
	void		(*irq_suspend)(struct irq_data *data);
	void		(*irq_resume)(struct irq_data *data);
	void		(*irq_pm_shutdown)(struct irq_data *data);

	void		(*irq_calc_mask)(struct irq_data *data);

	void		(*irq_print_chip)(struct irq_data *data, struct seq_file *p);
	int		(*irq_request_resources)(struct irq_data *data);
	void		(*irq_release_resources)(struct irq_data *data);

	void		(*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);
	void		(*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);

	int		(*irq_get_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool *state);
	int		(*irq_set_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool state);

	int		(*irq_set_vcpu_affinity)(struct irq_data *data, void *vcpu_info);

	void		(*ipi_send_single)(struct irq_data *data, unsigned int cpu);
	void		(*ipi_send_mask)(struct irq_data *data, const struct cpumask *dest);

	int		(*irq_nmi_setup)(struct irq_data *data);
	void		(*irq_nmi_teardown)(struct irq_data *data);

	unsigned long	flags;
};
```
