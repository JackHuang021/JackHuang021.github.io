---
title: Linux GPIO驱动框架
date: 2022-11-17 16:14:27
tags:
  - Linux
  - GPIO
categories: Linux
---

### 概述
Linux内核中对GPIO资源进行了抽象，抽象为gpiolib，管理GPIO资源

gpiolib汇总了GPIO的通用操作，根据GPIO的特性，gpiolib对上（其他用到GPIO的driver）提供一套统一的操作GPIO的软件接口，屏蔽了不同芯片的具体实现；对下（specific chip driver）提供了针对不同芯片操作的一套框架，针对不同芯片，只需要实现这套框架，然后使用gpiolib提供的注册函数，将其挂接到gpiolib上，就完成了gpio驱动的编写

### gpiolib相关数据结构
`struct gpio_chip`，这个结构是为了抽象GPIO的所有操作，这个结构是要开出去给其他芯片进行特定的操作赋值的，gpio_chip的抽象是针对GPIO一组bank的抽象，通常在硬件上，一个芯片的IO口，分为了很多个bank，每个bank分为了N组GPIO，可以理解为每一个bank有一组gpio操作的寄存器，可以通过这些寄存器对该bank的GPIO进行配置
```c
// include/linux/gpio/driver.h
struct gpio_chip {
	const char		*label;
	struct gpio_device	*gpiodev;
	struct device		*parent;
	struct module		*owner;

	int			(*request)(struct gpio_chip *chip,
						unsigned offset);
	void			(*free)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get_direction)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_input)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_output)(struct gpio_chip *chip,
						unsigned offset, int value);
	int			(*get)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get_multiple)(struct gpio_chip *chip,
						unsigned long *mask,
						unsigned long *bits);
	void			(*set)(struct gpio_chip *chip,
						unsigned offset, int value);
	void			(*set_multiple)(struct gpio_chip *chip,
						unsigned long *mask,
						unsigned long *bits);
	int			(*set_config)(struct gpio_chip *chip,
					      unsigned offset,
					      unsigned long config);
	int			(*to_irq)(struct gpio_chip *chip,
						unsigned offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *chip);
	int			base;
	u16			ngpio;
	const char		*const *names;
	bool			can_sleep;

#if IS_ENABLED(CONFIG_GPIO_GENERIC)
	unsigned long (*read_reg)(void __iomem *reg);
	void (*write_reg)(void __iomem *reg, unsigned long data);
	bool be_bits;
	void __iomem *reg_dat;
	void __iomem *reg_set;
	void __iomem *reg_clr;
	void __iomem *reg_dir;
	bool bgpio_dir_inverted;
	int bgpio_bits;
	spinlock_t bgpio_lock;
	unsigned long bgpio_data;
	unsigned long bgpio_dir;
#endif

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
#endif

	/**
	 * @need_valid_mask:
	 *
	 * If set core allocates @valid_mask with all bits set to one.
	 */
	bool need_valid_mask;

	/**
	 * @valid_mask:
	 *
	 * If not %NULL holds bitmask of GPIOs which are valid to be used
	 * from the chip.
	 */
	unsigned long *valid_mask;

#if defined(CONFIG_OF_GPIO)
	/*
	 * If CONFIG_OF is enabled, then all GPIO controllers described in the
	 * device tree automatically may have an OF translation
	 */

	/**
	 * @of_node:
	 *
	 * Pointer to a device tree node representing this GPIO controller.
	 */
	struct device_node *of_node;

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
#endif
};
```

`struct gpio_desc`结构体，每个GPIO由一个gpio_desc来描述，看起来`gpio_chip`和`gpio_desc`是包含关系，但是kernel中并没有直接将其两个结构联系上，而是通过另一个结构体将其联系在一起，这个结构就是`gpio_device`
```c
struct gpio_desc {
	struct gpio_device	*gdev;
	unsigned long		flags;
/* flag symbols are bit numbers */
#define FLAG_REQUESTED	0
#define FLAG_IS_OUT	1
#define FLAG_EXPORT	2	/* protected by sysfs_lock */
#define FLAG_SYSFS	3	/* exported via /sys/class/gpio/control */
#define FLAG_ACTIVE_LOW	6	/* value has active low */
#define FLAG_OPEN_DRAIN	7	/* Gpio is open drain type */
#define FLAG_OPEN_SOURCE 8	/* Gpio is open source type */
#define FLAG_USED_AS_IRQ 9	/* GPIO is connected to an IRQ */
#define FLAG_IS_HOGGED	11	/* GPIO is hogged */
#define FLAG_TRANSITORY 12	/* GPIO may lose value in sleep or reset */

	/* Connection label */
	const char		*label;
	/* Name of the GPIO */
	const char		*name;
};
```
gdev指针指向了这个gpio_desc所属的gpio_device，flag代表了该GPIO的属性状态

`struct gpio_device`结构，`gpio_device`是软件层面上对一个bank的GPIO进行管理的单元
```c
/**
 * struct gpio_device - internal state container for GPIO devices
 * @id: numerical ID number for the GPIO chip
 * @dev: the GPIO device struct
 * @chrdev: character device for the GPIO device
 * @mockdev: class device used by the deprecated sysfs interface (may be
 * NULL)
 * @owner: helps prevent removal of modules exporting active GPIOs
 * @chip: pointer to the corresponding gpiochip, holding static
 * data for this device
 * @descs: array of ngpio descriptors.
 * @ngpio: the number of GPIO lines on this GPIO device, equal to the size
 * of the @descs array.
 * @base: GPIO base in the DEPRECATED global Linux GPIO numberspace, assigned
 * at device creation time.
 * @label: a descriptive name for the GPIO device, such as the part number
 * or name of the IP component in a System on Chip.
 * @data: per-instance data assigned by the driver
 * @list: links gpio_device:s together for traversal
 *
 * This state container holds most of the runtime variable data
 * for a GPIO device and can hold references and live on after the
 * GPIO chip has been removed, if it is still being used from
 * userspace.
 */
struct gpio_device {
	int			id;
	struct device		dev;
	struct cdev		chrdev;
	struct device		*mockdev;
	struct module		*owner;
	struct gpio_chip	*chip;
	struct gpio_desc	*descs;
	int			base;
	u16			ngpio;
	const char		*label;
	void			*data;
	struct list_head        list;

#ifdef CONFIG_PINCTRL
	/*
	 * If CONFIG_PINCTRL is enabled, then gpio controllers can optionally
	 * describe the actual pin range which they serve in an SoC. This
	 * information would be used by pinctrl subsystem to configure
	 * corresponding pins for gpio usage.
	 */
	struct list_head pin_ranges;
#endif
};
```
`struct gpio_device`中包含了`struct gpio_chip`和`struct gpio_desc`，这个结构体贯穿了整个gpiolib，因为gpio_device是代表的一个bank，所以在kernel中，对gpio_device的组织是由一个gpio_device的链表构成的

### gpiolib对接芯片底层
底层需要对接的是实现对gpio的一些通用操作，这里主要是实现`gpio_chip`这个结构体，然后通过调用`gpiochip_add_data()`这个接口注册GPIO资源，或者使用`devm_gpiochip_add_data()`
```c
// include/linux/gpio/driver.h
#define gpiochip_add_data(chip, data) gpiochip_add_data_with_key(chip, data, NULL, NULL)

// drivers/gpio/gpiolib.c
int gpiochip_add_data_with_key(struct gpio_chip *chip, void *data,
			       struct lock_class_key *lock_key,
			       struct lock_class_key *request_key)
{
	unsigned long	flags;
	int		status = 0;
	unsigned	i;
	int		base = chip->base;
	struct gpio_device *gdev;

	/*
	 * First: allocate and populate the internal stat container, and
	 * set up the struct device.
	 */
    // 首先分配gpio_device结构体来管理gpio资源
    // 对gdev->chip进行赋值
    // 对chip->gpiodev进行赋值
	gdev = kzalloc(sizeof(*gdev), GFP_KERNEL);
	if (!gdev)
		return -ENOMEM;
	gdev->dev.bus = &gpio_bus_type;
	gdev->chip = chip;
	chip->gpiodev = gdev;
	if (chip->parent) {
		gdev->dev.parent = chip->parent;
		gdev->dev.of_node = chip->parent->of_node;
	}

#ifdef CONFIG_OF_GPIO
	/* If the gpiochip has an assigned OF node this takes precedence */
	if (chip->of_node)
		gdev->dev.of_node = chip->of_node;
	else
		chip->of_node = gdev->dev.of_node;
#endif

	gdev->id = ida_simple_get(&gpio_ida, 0, 0, GFP_KERNEL);
	if (gdev->id < 0) {
		status = gdev->id;
		goto err_free_gdev;
	}
	dev_set_name(&gdev->dev, "gpiochip%d", gdev->id);
	device_initialize(&gdev->dev);
	dev_set_drvdata(&gdev->dev, gdev);
	if (chip->parent && chip->parent->driver)
		gdev->owner = chip->parent->driver->owner;
	else if (chip->owner)
		/* TODO: remove chip->owner */
		gdev->owner = chip->owner;
	else
		gdev->owner = THIS_MODULE;

    // 分配ngpio个gpio_desc结构体
	gdev->descs = kcalloc(chip->ngpio, sizeof(gdev->descs[0]), GFP_KERNEL);
	if (!gdev->descs) {
		status = -ENOMEM;
		goto err_free_ida;
	}

	if (chip->ngpio == 0) {
		chip_err(chip, "tried to insert a GPIO chip with zero lines\n");
		status = -EINVAL;
		goto err_free_descs;
	}

	if (chip->ngpio > FASTPATH_NGPIO)
		chip_warn(chip, "line cnt %u is greater than fast path cnt %u\n",
		chip->ngpio, FASTPATH_NGPIO);

	gdev->label = kstrdup_const(chip->label ?: "unknown", GFP_KERNEL);
	if (!gdev->label) {
		status = -ENOMEM;
		goto err_free_descs;
	}

	gdev->ngpio = chip->ngpio;
    // 将GPIO驱动指针存放到gpio_device->data中
	gdev->data = data;

	spin_lock_irqsave(&gpio_lock, flags);

	/*
	 * TODO: this allocates a Linux GPIO number base in the global
	 * GPIO numberspace for this chip. In the long run we want to
	 * get *rid* of this numberspace and use only descriptors, but
	 * it may be a pipe dream. It will not happen before we get rid
	 * of the sysfs interface anyways.
	 */
    // base代表每个bank的编号
	if (base < 0) {
		base = gpiochip_find_base(chip->ngpio);
		if (base < 0) {
			status = base;
			spin_unlock_irqrestore(&gpio_lock, flags);
			goto err_free_label;
		}
		/*
		 * TODO: it should not be necessary to reflect the assigned
		 * base outside of the GPIO subsystem. Go over drivers and
		 * see if anyone makes use of this, else drop this and assign
		 * a poison instead.
		 */
		chip->base = base;
	}
	gdev->base = base;

    // 将该gpio_device挂到gpio_devices链表中进行管理
	status = gpiodev_add_to_list(gdev);
	if (status) {
		spin_unlock_irqrestore(&gpio_lock, flags);
		goto err_free_label;
	}

	spin_unlock_irqrestore(&gpio_lock, flags);

    // 初始化该组bank中每个gpio的gpio_desc的信息
	for (i = 0; i < chip->ngpio; i++) {
		struct gpio_desc *desc = &gdev->descs[i];

		desc->gdev = gdev;

		/* REVISIT: most hardware initializes GPIOs as inputs (often
		 * with pullups enabled) so power usage is minimized. Linux
		 * code should set the gpio direction first thing; but until
		 * it does, and in case chip->get_direction is not set, we may
		 * expose the wrong direction in sysfs.
		 */
		desc->flags = !chip->direction_input ? (1 << FLAG_IS_OUT) : 0;
	}

#ifdef CONFIG_PINCTRL
	INIT_LIST_HEAD(&gdev->pin_ranges);
#endif

	status = gpiochip_set_desc_names(chip);
	if (status)
		goto err_remove_from_list;

	status = gpiochip_irqchip_init_valid_mask(chip);
	if (status)
		goto err_remove_from_list;

	status = gpiochip_init_valid_mask(chip);
	if (status)
		goto err_remove_irqchip_mask;

	status = gpiochip_add_irqchip(chip, lock_key, request_key);
	if (status)
		goto err_remove_chip;

	status = of_gpiochip_add(chip);
	if (status)
		goto err_remove_chip;

	acpi_gpiochip_add(chip);

	machine_gpiochip_add(chip);

	/*
	 * By first adding the chardev, and then adding the device,
	 * we get a device node entry in sysfs under
	 * /sys/bus/gpio/devices/gpiochipN/dev that can be used for
	 * coldplug of device nodes and other udev business.
	 * We can do this only if gpiolib has been initialized.
	 * Otherwise, defer until later.
	 */
	if (gpiolib_initialized) {
		status = gpiochip_setup_dev(gdev);
		if (status)
			goto err_remove_chip;
	}
	return 0;

err_remove_chip:
	acpi_gpiochip_remove(chip);
	gpiochip_free_hogs(chip);
	of_gpiochip_remove(chip);
	gpiochip_free_valid_mask(chip);
err_remove_irqchip_mask:
	gpiochip_irqchip_free_valid_mask(chip);
err_remove_from_list:
	spin_lock_irqsave(&gpio_lock, flags);
	list_del(&gdev->list);
	spin_unlock_irqrestore(&gpio_lock, flags);
err_free_label:
	kfree_const(gdev->label);
err_free_descs:
	kfree(gdev->descs);
err_free_ida:
	ida_simple_remove(&gpio_ida, gdev->id);
err_free_gdev:
	/* failures here can mean systems won't boot... */
	pr_err("%s: GPIOs %d..%d (%s) failed to register, %d\n", __func__,
	       gdev->base, gdev->base + gdev->ngpio - 1,
	       chip->label ? : "generic", status);
	kfree(gdev);
	return status;
}
EXPORT_SYMBOL_GPL(gpiochip_add_data_with_key);
```

### 对GPIO中断的管理
`struct gpio_irq_chip`结构体，GPIO中断控制器
```c
// include/linux/gpio/driver.h
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

	/**
	 * @domain_ops:
	 *
	 * Table of interrupt domain operations for this IRQ chip.
	 */
	const struct irq_domain_ops *domain_ops;

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
	 * Per GPIO IRQ chip lockdep classes.
	 */
	struct lock_class_key *lock_key;
	struct lock_class_key *request_key;

	/**
	 * @parent_handler:
	 *
	 * The interrupt handler for the GPIO chip's parent interrupts, may be
	 * NULL if the parent interrupts are nested rather than cascaded.
	 */
	irq_flow_handler_t parent_handler;

	/**
	 * @parent_handler_data:
	 *
	 * Data associated, and passed to, the handler for the parent
	 * interrupt.
	 */
	void *parent_handler_data;

	/**
	 * @num_parents:
	 *
	 * The number of interrupt parents of a GPIO chip.
	 */
	unsigned int num_parents;

	/**
	 * @parent_irq:
	 *
	 * For use by gpiochip_set_cascaded_irqchip()
	 */
	unsigned int parent_irq;

	/**
	 * @parents:
	 *
	 * A list of interrupt parents of a GPIO chip. This is owned by the
	 * driver, so the core will only reference this list, not modify it.
	 */
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
	 * @need_valid_mask:
	 *
	 * If set core allocates @valid_mask with all bits set to one.
	 */
	bool need_valid_mask;

	/**
	 * @valid_mask:
	 *
	 * If not %NULL holds bitmask of GPIOs which are valid to be included
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
};
```

#### phytium gpio驱动
phytium_gpio结构体定义
```c
struct phytium_gpio {
	raw_spinlock_t		lock;
	void __iomem		*regs;
	// 用来表示一个gpio控制器
	struct gpio_chip	gc;

	struct irq_chip		irq_chip;
	// 用来表示gpio个数
	unsigned int		ngpio[2];
	// gpio中断号数组
	int			irq[32];
#ifdef CONFIG_PM_SLEEP
	struct phytium_gpio_ctx	ctx;
#endif
};
```
例如Phytium E2000 CPU共6个GPIO控制器，每个控制器提供16个GPIO信号  
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230506100520.png)

### Linux中断通用框架
在linux中引入了irq_domain的概念，一个中断控制器就是一个irq_domain，中断控制器出现级联的布局，利用树状的结构可以充分的利用irq数目，每一个irq_domain可以去管理自己interrupt的特性

#### IRQ domain
`irq_domain`用于将硬件的中断号，转换为Linux系统中的中断号（virq）
+ 每个中断控制器都对应一个irq_domain
+ 中断控制器驱动通过irq_domain_add()接口来创建irq_domain
+ irq_domain支持三种映射方式：linear map，tree map，no map
	+ linear map：线性映射，维护固定大小的表，索引是硬件中断号，如果硬件中断最大数量固定，数值不大，可以选择线性映射
	+ tree map：硬件中断号可能很大，可以选择树映射
	+ no map：硬件中断号直接就是linux的中断号
+ 各个控制器的硬件中断号可以一样，最终在Linux内核中映射的中断号是唯一的

每一个中断控制器对应多个中断号，而硬件中断号在不同的中断控制器上是会重复编码的，因此linux kernel提供了一个虚拟中断号的概念

#### 与中断相关的数据结构
在看硬件中断号映射到虚拟中断号之前，来看几个比较重要的数据结构

`struct irq_chip`结构体描述中断控制器的底层操作函数集，这些函数集最终完成对控制器硬件的操作
```c
struct irq_chip {
	struct device	*parent_device;
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

	void		(*irq_cpu_online)(struct irq_data *data);
	void		(*irq_cpu_offline)(struct irq_data *data);

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

`struct irq_desc`描述一个外设的中断，称为中断描述符
```c
struct irq_desc {
	struct irq_common_data	irq_common_data;
	// 中断控制器的硬件数据
	struct irq_data		irq_data;
	unsigned int __percpu	*kstat_irqs;
	// 中断控制器驱动的处理函数
	irq_flow_handler_t	handle_irq;
	struct irqaction	*action;	/* IRQ action list */
	unsigned int		status_use_accessors;
	unsigned int		core_internal_state__do_not_mess_with_it;
	unsigned int		depth;		/* nested irq disables */
	unsigned int		wake_depth;	/* nested wake enables */
	unsigned int		tot_count;
	unsigned int		irq_count;	/* For detecting broken IRQs */
	unsigned long		last_unhandled;	/* Aging timer for unhandled count */
	unsigned int		irqs_unhandled;
	atomic_t		threads_handled;
	int			threads_handled_last;
	raw_spinlock_t		lock;
	struct cpumask		*percpu_enabled;
	const struct cpumask	*percpu_affinity;
#ifdef CONFIG_SMP
	const struct cpumask	*affinity_hint;
	struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
	cpumask_var_t		pending_mask;
#endif
#endif
	unsigned long		threads_oneshot;
	atomic_t		threads_active;
	wait_queue_head_t       wait_for_threads;
#ifdef CONFIG_PM_SLEEP
	unsigned int		nr_actions;
	unsigned int		no_suspend_depth;
	unsigned int		cond_suspend_depth;
	unsigned int		force_resume_depth;
#endif
#ifdef CONFIG_PROC_FS
	struct proc_dir_entry	*dir;
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
	struct dentry		*debugfs_file;
	const char		*dev_name;
#endif
#ifdef CONFIG_SPARSE_IRQ
	struct rcu_head		rcu;
	struct kobject		kobj;
#endif
	struct mutex		request_mutex;
	int			parent_irq;
	struct module		*owner;
	const char		*name;
} ____cacheline_internodealigned_in_smp;
```

`struct irq_data`包含中断控制器的硬件数据
```c
struct irq_data {
	u32			mask;
	// 虚拟中断号
	unsigned int		irq;
	// 硬件中断号
	unsigned long		hwirq;
	struct irq_common_data	*common;
	// 对应的irq_chip数据结构
	struct irq_chip		*chip;
	// 对应的irq_domain数据结构
	struct irq_domain	*domain;
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	struct irq_data		*parent_data;
#endif
	void			*chip_data;
};
```

`struct irq_domain`与中断控制器对应，完成硬件中断hwirq到virq的映射
```c
struct irq_domain {
	// 用于将irq_domain链接到全局链表irq_domain_list中
	struct list_head link;
	const char *name;
	// irq_domain映射操作函数集
	const struct irq_domain_ops *ops;
	// 在GIC中断控制器驱动中，指向了irq_dic_data
	void *host_data;
	unsigned int flags;
	// 映射中断的个数
	unsigned int mapcount;

	/* Optional data */
	struct fwnode_handle *fwnode;
	enum irq_domain_bus_token bus_token;
	struct irq_domain_chip_generic *gc;
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	// 支持中断级联的话，指向父设备
	struct irq_domain *parent;
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
	struct dentry		*debugfs_file;
#endif

	/* reverse map data. The linear map gets appended to the irq_domain */
	// irq_domain支持中断数量的最大值
	irq_hw_number_t hwirq_max;
	unsigned int revmap_direct_max_irq;
	// 线性映射的大小
	unsigned int revmap_size;
	struct radix_tree_root revmap_tree;
	struct mutex revmap_tree_mutex;
	// 线性映射用到的查找表
	unsigned int linear_revmap[];
};
```

`struct irq_domain`结构体，用于irq_domain映射操作的函数集
```c
struct irq_domain_ops {
	// 用于中断控制器和irq domain的匹配
	int (*match)(struct irq_domain *d, struct device_node *node,
		     enum irq_domain_bus_token bus_token);
	int (*select)(struct irq_domain *d, struct irq_fwspec *fwspec,
		      enum irq_domain_bus_token bus_token);
	// 用于硬件中断号和linux中断号的映射
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

`struct irqaction`用来存储设备驱动注册的中断处理函数
```c
struct irqaction {
	// 设备驱动里的中断处理函数
	irq_handler_t		handler;
	// 设备id
	void			*dev_id;
	void __percpu		*percpu_dev_id;
	struct irqaction	*next;
	irq_handler_t		thread_fn;
	struct task_struct	*thread;
	struct irqaction	*secondary;
	// 中断号
	unsigned int		irq;
	// 中断标志
	unsigned int		flags;
	unsigned long		thread_flags;
	unsigned long		thread_mask;
	const char		*name;
	struct proc_dir_entry	*dir;
} ____cacheline_internodealigned_in_smp;
```

其核心是围绕着`struct irq_desc`来展开的
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230506140748.png)

中断的处理主要有以下几个功能模块
+ 硬件中断号到linux irq中断号的映射，并创建好irq_desc中断描述符
+ 中断注册时，先获取设备的中断号，根据中断号找到对应的irq_desc，并将设备的中断处理函数添加到irq_desc
+ 设备触发中断信号时，根据硬件中断号得到linux irq中断号，找到对应的irq_desc，最终调用到设备的中断处理函数


##### 如何取得linux irq中断号
硬件设备的中断信息都在设备树中进行了描述，在系统启动过程中，这些信息都已经解析并加载到了内存中，驱动中通常会使用`platform_get_irq()`接口，去根据设备树的信息去创建映射关系，硬件中断号到linux irq中断号映射，



```bash
setenv bootargs console=ttyAMA1,115200 earlycon=pl011,0x28001000 root=/dev/sda rootwait rw;ext4load scsi 0:0 0x90100000 boot/d2000/d2000-devboard-dsk.dtb;ext4load scsi 0:0 0x90200000 boot/d2000/Image;booti 0x90200000 - 0x90100000


108:          0          0  28036000.gpio   5 Edge      es8336_interrupt

109:          0          0  28038000.gpio  11 Edge      gpiomon
```







