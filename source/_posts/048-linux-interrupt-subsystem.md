---
title: Linux中断子系统
tags:
  - Linux
  - Interrupt
categories: Linux
abbrlink: 6ea84d7b
date: 2023-08-17 17:46:52
---

### 中断子系统的一些概念
#### IRQ Domain介绍

IRQ number： CPU需要为每一个外设中断编号，这里称之为IRQ Number，这个IRQ Number是一个虚拟的interrupt id，和硬件无关，仅仅是被CPU用来标识一个外设中断

HW interrupt ID：对于interrupt controller而言，它收集了多个外设的interrupt request line并向上传递，所以interrupt controller需要对外设中断进行编码，interrupt controller使用HW interrupt ID来标识外设的中断，

对于驱动工程师而言，我们只希望会得到一个IRQ number，而不关心具体是那个interrupt controller上的哪个HW Interrupt ID，因此linux kernel中的中断子系统需要提供一个将HW interrupt ID映射到IRQ number上来的机制

#### 为IRQ domain创建映射


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

