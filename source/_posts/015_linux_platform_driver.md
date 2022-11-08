---
title: Platform设备驱动
author: Jack
tags:
  - Linux
  - Platform
categories:
  - Linux
abbrlink: f8def83d
---

#### Platform平台驱动模型
+ 由于某些外设是没有总线这个概念的，但是又要使用总线、驱动和设备模型的话，就需要使用`platform`这个虚拟总线，相应的就有`platform_device`和`platform_driver`
<!-- more -->

#### Platform总线
+ Linux内核使用`bus_type`结构体表示总线，该结构体定义在`include/linux/device.h`
    ```c
    struct bus_type {
        const char		*name;
        const char		*dev_name;
        struct device		*dev_root;
        const struct attribute_group **bus_groups;
        const struct attribute_group **dev_groups;
        const struct attribute_group **drv_groups;

        int (*match)(struct device *dev, struct device_driver *drv);
        int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
        int (*probe)(struct device *dev);
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
    设备和驱动的匹配函数：
    `int (*match)(struct device *dev, struct device_driver*drv)`
    match函数有两个参数dev和driver，这两个参数分别为`device`和`device_driver`类型，也就是设备和驱动

+ platform总线是`bus_type`的一个具体实例，定义在`driver/base/platform.c`中
    ```c
    struct bus_type platform_bus_type = {
        .name		= "platform",
        .dev_groups	= platform_dev_groups,
        .match		= platform_match,
        .uevent		= platform_uevent,
        .dma_configure	= platform_dma_configure,
        .pm		= &platform_dev_pm_ops,
    };
    ```
    其中`platform_match`为驱动和设备的匹配函数

+ `platform_match`函数如下
    ```c
    static int platform_match(struct device *dev, struct device_driver *drv)
    {
        struct platform_device *pdev = to_platform_device(dev);
        struct platform_driver *pdrv = to_platform_driver(drv);

        /* When driver_override is set, only bind to the matching driver */
        if (pdev->driver_override)
            return !strcmp(pdev->driver_override, drv->name);

        /* Attempt an OF style match first */
        if (of_driver_match_device(dev, drv))
            return 1;

        /* Then try ACPI style match */
        if (acpi_driver_match_device(dev, drv))
            return 1;

        /* Then try to match against the id table */
        if (pdrv->id_table)
            return platform_match_id(pdrv->id_table, pdev) != NULL;

        /* fall-back to driver name match */
        return (strcmp(pdev->name, drv->name) == 0);
    }
    ```
    platform驱动和设备匹配一共有四种方法：
    + OF类型的匹配，也就是采用设备树的方式，`of_driver_match_device()`定义在文件 `/include/linux/of_device`中，`device_drive`结构体中有一个名为`of_match_table`的成员变量，该成员保存着驱动的compatible匹配表，设备树中的每个设备节点的compatible属性会和of_match_table中的所有成员比较，查看是否有相同的条目，如果有的话表示设备和驱动相匹配，设备和驱动匹配成功后，probe函数就会执行
    + ACPI匹配方式
    + `id_table`匹配，每个`platform_driver`结构体有一个`id_table`成员变量，保留了很多id信息，这些id信息存放着这个驱动所支持的设备信息，该`id_table`会与`platform_device`中的`name`成员相比较
    + 如果`id_table`不存在的话就直接比较驱动和设备的`name`字段，看看是否相等

#### Platform总线初始化过程
+ 内核在初始化过程中调用`platform_bus_init()`来初始化Platform总线，调用流程如下  
`kernel_init_freeable() -> do_basic_setup() -> driver_init() -> platform_bus_init()`  
其中`platform_bus_init()`函数定义在`driver/base/platform.c`中，内容如下：
    ```c
    int __init platform_bus_init(void)
    {
        int error;

        early_platform_cleanup();

        error = device_register(&platform_bus);
        if (error) {
            put_device(&platform_bus);
            return error;
        }
        error =  bus_register(&platform_bus_type);
        if (error)
            device_unregister(&platform_bus);
        of_platform_register_reconfig_notifier();
        return error;
    }
    ```
    + 先清空总线`early_platform_device_list`上的所有节点，`early_platform_device_list`定义如下  
    `static __initdata LIST_HEAD(early_platform_device_list);` 

#### Platform驱动
+ `platform_driver`结构体表示`platform`驱动，定义在`include/linux/platform.h`里面
    ```c
    struct platform_driver {
        int (*probe)(struct platform_device *);
        int (*remove)(struct platform_device *);
        void (*shutdown)(struct platform_device *);
        int (*suspend)(struct platform_device *, pm_message_t state);
        int (*resume)(struct platform_device *);
        struct device_driver driver;
        const struct platform_device_id *id_table;
        bool prevent_deferred_probe;
    };
    ```
    当驱动和设备匹配成功后`platform_driver`的`probe`函数就会执行，`driver`成员为`device_driver`结构体变量，`device_driver`相当于基类，提供了最基础的驱动框架，`platform_driver`相当于继承了这个基类，在这个基类基础上添加了一些特有的成员变量

+ `device_driver`结构体定义在*`include/linux/device.h`，内容如下
    ```c
    struct device_driver {
        const char		*name;
        struct bus_type		*bus;

        struct module		*owner;
        const char		*mod_name;	/* used for built-in modules */

        bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
        enum probe_type probe_type;

        const struct of_device_id	*of_match_table;
        const struct acpi_device_id	*acpi_match_table;

        int (*probe) (struct device *dev);
        int (*remove) (struct device *dev);
        void (*shutdown) (struct device *dev);
        int (*suspend) (struct device *dev, pm_message_t state);
        int (*resume) (struct device *dev);
        const struct attribute_group **groups;

        const struct dev_pm_ops *pm;
        void (*coredump) (struct device *dev);

        struct driver_private *p;
    };
    ```

+ 在编写`platform`驱动的时候，首先需要定义一个`platform_driver`结构体变量，实现结构体中的成员变量，重点是实现匹配方法和`probe`函数，具体的驱动程序在`probe`里面编写

+ 定义好`platform_driver`结构体变量以后，需要在驱动入口函数里面调用`platform_driver_register()`函数向内核注册一个platform驱动，         `platform_driver_register()`函数的原型如下：
`int platform_driver_register(struct platform_driver *driver)`
还需要在驱动卸载函数中通过`platform_driver_unregister()`来卸载platform驱动，函数原型如下：
`int platform_driver_unregister(struct platform_driver *driver)`

#### Platform设备
+ `platform_device`结构体表示`platfrom`设备，如果内核支持设备树的话，就不需要使用`platform_device`来描述设备，改用设备树来描述，`platform_device`结构体定义在*include/linux/platform_device.h*中，结构体的内容如下：
    ```c
    struct platform_device {
        const char	*name;
        int		id;
        bool		id_auto;
        struct device	dev;
        u32		num_resources;
        struct resource	*resource;

        const struct platform_device_id	*id_entry;
        char *driver_override; /* Driver name to force a match */

        /* MFD cell pointer */
        struct mfd_cell *mfd_cell;

        /* arch specific additions */
        struct pdev_archdata	archdata;
    };
    ```
    + `name`表示设备的名字，要和所使用的`platform_driver`的`name`字段相同，否则设备无法匹配到对应的驱动
    + `resource`表示资源，一般用来表示设备的寄存器信息，结构体内容如下
    ```c
    struct resource {
        resource_size_t start;
        resource_size_t end;
        const char *name;
        unsigned long flags;
        unsigned long desc;
        struct resource *parent, *sibling, *child;
    };
    ```
    `start`和`end`分别表示资源的起始和终止信息，对于内存类资源就表示内存起始和终止地址，`name`表示资源名称，`flag`表示资源类型

+ Linux支持设备树之后就不需要用户去手动注册`platform`设备了，因为设备信息都放到设备树中去描述，Linux内核启动的时候会从设备树中读取设备信息，然后将其组织成`platform_device`形式

#### Platform驱动匹配过程
+ 这里使用phytium i2c适配器驱动分析，驱动文件位于`drivers/i2c/busses/i2c-phytium-platform.c`，在`module_platform_driver(phytium_i2c_driver)`进行`platform_drvier`的注册，`module_platform_driver`是一个宏定义，该宏定义定义在`include/linux/platform_device.h`其内容如下：
    ```c  
    #define module_platform_driver(__platform_drvier) module_driver(__platform_drvier, platform_drvier_register, platform_drvier_unregister)
    ```
    继续对*module_driver*宏进行展开，其内容如下：  
    ```c
    #define module_driver(__driver, __register, __unregister, ...) \
    static int __init __driver##_init(void) \
    { \
        return __register(&(__driver) , ##__VA_ARGS__); \
    } \
    module_init(__driver##_init); \
    static void __exit __driver##_exit(void) \
    { \
        __unregister(&(__driver) , ##__VA_ARGS__); \
    } \
    module_exit(__driver##_exit);
    ```
    最终展开后的内容相当于：
    ```c
    static int __init phytium_i2c_driver_init(void)
    {
        return platform_driver_register(&(phytium_i2c_driver));
    }
    module_init(phytium_i2c_driver_init);
    static int __exit phytium_i2c_driver_exit(void)
    {
        return platform_driver_unregister(&(phytium_i2c_driver));
    }
    moudle_exit(phytium_i2c_driver_exit);
    ```

+ 在`moudle_init`中调用了`platform_drvier_register(&phytium_i2c_driver)`，`platform_driver_regsiter`又是一个宏定义，定义在`include/linux/platform_device.h`中，其内容如下：  
`#define platform_driver_regsiter(drv) __platform_drvier_register(drv, THIS_MODULE)`  
其中`__platform_drvier_register()`函数的定义如下：  
    ```c
    int __platform_drvier_register(struct platform_deriver *drv, struct module *owner)`
    {
        drv->driver.owner = owner;
        drv->driver.bus = &platform_bus_type;
        drv->driver.probe = platform_drv_porbe;
        drv->driver.remove = platform_drv_remove;
        drv->driver.shutdown = platform_drv_shutdown;

        return driver_register(&drv->driver);
    }
    ```
    其中`driver`成员的类型为之前提到的`device_driver`结构体，`driver->bus`指向了`platform_bus_type`


+ 继续来看`driver_register(&drv->driver)`，该函数位于`driver/base/driver.c`中，其内容如下
    ```c
    int driver_register(struct device_driver *drv)
    {
        int ret;
        struct device_driver *other;

        if (!drv->bus->p) {
            pr_err("Driver '%s' was unable to register with bus_type '%s' because the bus was not initialized.\n",
                drv->name, drv->bus->name);
            return -EINVAL;
        }
        /* 检测总线的操作函数和驱动的操作函数是否都已经定义好 */
        if ((drv->bus->probe && drv->probe) ||
            (drv->bus->remove && drv->remove) ||
            (drv->bus->shutdown && drv->shutdown))
            printk(KERN_WARNING "Driver '%s' needs updating - please use "
                "bus_type methods\n", drv->name);

        other = driver_find(drv->name, drv->bus);
        if (other) {
            printk(KERN_ERR "Error: Driver '%s' is already registered, "
                "aborting...\n", drv->name);
            return -EBUSY;
        }

        ret = bus_add_driver(drv);
        if (ret)
            return ret;
        ret = driver_add_groups(drv, drv->groups);
        if (ret) {
            bus_remove_driver(drv);
            return ret;
        }
        kobject_uevent(&drv->p->kobj, KOBJ_ADD);

        return ret;
    }
    ```
    `driver_register()`中调用了`bus_add_driver(drv)`，该函数位于`driver/base/bus.c`中，`bus_add_driver()`源码如下：
    ```c
    /**
     * bus_add_driver - Add a driver to the bus.
     * @drv: driver.
     */
    int bus_add_driver(struct device_driver *drv)
    {
    	struct bus_type *bus;
    	struct driver_private *priv;
    	int error = 0;

    	bus = bus_get(drv->bus);
    	if (!bus)
    		return -EINVAL;

    	pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);

    	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
    	if (!priv) {
    		error = -ENOMEM;
    		goto out_put_bus;
    	}
    	klist_init(&priv->klist_devices, NULL, NULL);
    	priv->driver = drv;
    	drv->p = priv;
    	priv->kobj.kset = bus->p->drivers_kset;
    	error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
    				     "%s", drv->name);
    	if (error)
    		goto out_unregister;

    	klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
    	if (drv->bus->p->drivers_autoprobe) {
    		error = driver_attach(drv);
    		if (error)
    			goto out_unregister;
    	}
    	module_add_driver(drv->owner, drv);

    	error = driver_create_file(drv, &driver_attr_uevent);
    	if (error) {
    		printk(KERN_ERR "%s: uevent attr (%s) failed\n",
    			__func__, drv->name);
    	}
    	error = driver_add_groups(drv, bus->drv_groups);
    	if (error) {
    		/* How the hell do we get out of this pickle? Give up */
    		printk(KERN_ERR "%s: driver_create_groups(%s) failed\n",
    			__func__, drv->name);
    	}

    	if (!drv->suppress_bind_attrs) {
    		error = add_bind_files(drv);
    		if (error) {
    			/* Ditto */
    			printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
    				__func__, drv->name);
    		}
    	}

    	return 0;

    out_unregister:
    	kobject_put(&priv->kobj);
    	/* drv->p is freed in driver_release()  */
    	drv->p = NULL;
    out_put_bus:
    	bus_put(bus);
    	return error;
    }


    ```





