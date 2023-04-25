---
title: Platform设备驱动模型
author: Jack
tags:
  - Linux
  - Platform
categories:
  - Linux
abbrlink: f8def83d
---

#### Platform平台驱动模型
Linux内核在2.6版本引入设备驱动模型，简化了驱动程序的编写，Linux设备驱动模型包含**设备（device）、总线（bus）、类（class）和驱动（driver）**，其中**设备**和**驱动**通过**总线**绑定在一起，由于某些外设是没有总线这个概念的，但是又要使用总线、驱动和设备模型的话，就需要使用`platform`这个虚拟总线，相应的就有`platform_device`和`platform_driver`

<!-- more -->

Linux内核中，分别用`bus_type`、`device_driver`和`device`结构体来描述总线、驱动和设备，结构体定义在`include/linux/device.h`，设备和对应的驱动必须依附于同一种总线，因此`device_driver`和`device`结构体中都包含`bus_type`指针


#### Platform总线
Linux内核使用`bus_type`结构体表示总线，该结构体定义在`include/linux/device/bus.h`
```c
// include/linux/device/bus.h
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

platform总线是`bus_type`的一个具体实例，定义在`driver/base/platform.c`中，同时也定义了设备`platform_bus`，用来管理所有挂载在platform总线下的设备，定义如下：
```c
// driver/base/platform.c
struct bus_type platform_bus_type = {
    .name		= "platform",
    .dev_groups	= platform_dev_groups,
    .match		= platform_match,
    .uevent		= platform_uevent,
    .dma_configure	= platform_dma_configure,
    .pm		= &platform_dev_pm_ops,
};

struct device platform_bus = {
    .init_name = "platform",
};
```

##### Platform总线初始化过程
内核在初始化过程中调用`platform_bus_init()`来初始化Platform总线，调用流程如下  
`kernel_init_freeable() -> do_basic_setup() -> driver_init() -> platform_bus_init()`  
其中`platform_bus_init()`函数定义在`drivers/base/platform.c`中，内容如下：
```c
// drivers/base/platform.c
int __init platform_bus_init(void)
{
    int error;

    early_platform_cleanup();

    error = device_register(&platform_bus);
    if (error) {
        put_device(&platform_bus);
        return error;
    }
    error = bus_register(&platform_bus_type);
    if (error)
        device_unregister(&platform_bus);
    of_platform_register_reconfig_notifier();
    return error;
}
```
首先来看`early_platform_cleanup()`这个函数，位于`arch/sh/drivers/platform_early.c`中，这个函数主要的功能就是清除所有和early device/driver相关的代码，执行到这里的时候，证明系统已经完成了early阶段的启动，转而进行正常的设备初始化、启动操作，所以不再需要early platform相关的东西

```c
// include/linux/init.h
#define __initdata __section(".init.data")

static _initdata LIST_HEAD(sh_early_platform_device_list);

/**
 * early_platform_cleanup - clean up early platform code
 */
void __init early_platform_cleanup(void)
{
    struct platform_device *pd, *pd2;

    /* clean up the devres list used to chain devices */
    list_for_each_entry_safe(pd, pd2, &sh_early_platform_device_list,
                        dev.devres_head) {
        list_del(&pd->dev.devres_head);
        memset(&pd->dev.devres_head, 0, sizeof(pd->dev.devres_head));
    }
}
```

接着来看`device_register(&platform_bus)`，将`platform_bus`设备注册到驱动模型中，该步骤会在sysfs中创建`/sys/device/platform/`目录，所有的platform设备都会包含在此目录下，所有的platform设备都会包含在此目录下
```c
// drivers/base/core.c
/**
 * device_register - register a device with the system.
 * @dev: pointer to the device structure
 *
 * This happens in two clean steps - initialize the device
 * and add it to the system. The two steps can be called
 * separately, but this is the easiest and most common.
 * I.e. you should only call the two helpers separately if
 * have a clearly defined need to use and refcount the device
 * before it is added to the hierarchy.
 *
 * For more information, see the kerneldoc for device_initialize()
 * and device_add().
 *
 * NOTE: _Never_ directly free @dev after calling this function, even
 * if it returned an error! Always use put_device() to give up the
 * reference initialized in this function instead.
 */
int device_register(struct device *dev)
{
    device_initialize(dev);
    return device_add(dev);
}
```

然后是`bus_register(&platform_bus_type);`，将`platform`总线注册到Linux的总线系统中
```c
// drivers/base/base.h
// 子系统私有数据的数据结构定义
/**
 * struct subsys_private - structure to hold the private to the driver core portions of the bus_type/class structure.
 *
 * @subsys - the struct kset that defines this subsystem
 * @devices_kset - the subsystem's 'devices' directory
 * @interfaces - list of subsystem interfaces associated
 * @mutex - protect the devices, and interfaces lists.
 *
 * @drivers_kset - the list of drivers associated
 * @klist_devices - the klist to iterate over the @devices_kset
 * @klist_drivers - the klist to iterate over the @drivers_kset
 * @bus_notifier - the bus notifier list for anything that cares about things
 *                 on this bus.
 * @bus - pointer back to the struct bus_type that this structure is associated
 *        with.
 *
 * @glue_dirs - "glue" directory to put in-between the parent device to
 *              avoid namespace conflicts
 * @class - pointer back to the struct class that this structure is associated
 *          with.
 *
 * This structure is the one that is the actual kobject allowing struct
 * bus_type/class to be statically allocated safely.  Nothing outside of the
 * driver core should ever touch these fields.
 */
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

    // 初始化bus的私有数据
	priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

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

	retval = bus_create_file(bus, &bus_attr_uevent);
	if (retval)
		goto bus_uevent_fail;

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

	INIT_LIST_HEAD(&priv->interfaces);
	__mutex_init(&priv->mutex, "subsys mutex", key);
	klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
	klist_init(&priv->klist_drivers, NULL, NULL);

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

#### Platform驱动
`platform_driver`结构体表示`platform`驱动，定义在`include/linux/platform.h`里面
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

`device_driver`结构体定义在*`include/linux/device.h`，内容如下
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

在编写`platform`驱动的时候，首先需要定义一个`platform_driver`结构体变量，实现结构体中的成员变量，重点是实现匹配方法和`probe`函数，具体的驱动程序在`probe`里面编写

定义好`platform_driver`结构体变量以后，需要在驱动入口函数里面调用`platform_driver_register()`函数向内核注册一个platform驱动，`platform_driver_register()`函数的原型如下：
`int platform_driver_register(struct platform_driver *driver)`

还需要在驱动卸载函数中通过`platform_driver_unregister()`来卸载platform驱动，函数原型如下：
`int platform_driver_unregister(struct platform_driver *driver)`

#### Platform设备
`platform_device`结构体表示`platfrom`设备，如果内核支持设备树的话，就不需要使用`platform_device`来描述设备，改用设备树来描述，`platform_device`结构体定义在*include/linux/platform_device.h*中，结构体的内容如下：
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

Linux支持设备树之后就不需要用户去手动注册`platform`设备了，因为设备信息都放到设备树中去描述，Linux内核启动的时候会从设备树中读取设备信息，然后将其组织成`platform_device`形式

#### Platform驱动匹配过程
这里使用phytium i2c适配器驱动分析，驱动文件位于`drivers/i2c/busses/i2c-phytium-platform.c`，在`module_platform_driver(phytium_i2c_driver)`进行`platform_drvier`的注册，`module_platform_driver`是一个宏定义，该宏定义定义在`include/linux/platform_device.h`其内容如下：
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

在`moudle_init`中调用了`platform_drvier_register(&phytium_i2c_driver)`，`platform_driver_regsiter`又是一个宏定义，定义在`include/linux/platform_device.h`中，其内容如下：  
`#define platform_driver_regsiter(drv) __platform_drvier_register(drv, THIS_MODULE)`  
其中`__platform_drvier_register()`函数的定义如下：  
```c
// drivers/base/platform.c
int __platform_drvier_register(struct platform_driver *drv, struct module *owner)`
{
    drv->driver.owner = owner;
    // bus_type指向platform_bus_type
    drv->driver.bus = &platform_bus_type;
    drv->driver.probe = platform_drv_porbe;
    drv->driver.remove = platform_drv_remove;
    drv->driver.shutdown = platform_drv_shutdown;
    // 继续调用device_driver注册设备驱动
    return driver_register(&drv->driver);
}
```
其中`driver`成员的类型为之前提到的`device_driver`结构体，`driver->bus`指向了`platform_bus_type`


继续来看`driver_register(&drv->driver)`，该函数位于`drivers/base/driver.c`中，其内容如下
```c
// drivers/base/driver.c
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

    // 检查driver是否已经在bus上注册
    // 已经注册的话会返回device_driver结构
    other = driver_find(drv->name, drv->bus);
    if (other) {
        printk(KERN_ERR "Error: Driver '%s' is already registered, "
            "aborting...\n", drv->name);
        return -EBUSY;
    }
    // 将该device_driver注册到总线
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
    */,1
int bus_add_driver(struct device_driver *drv)
{
    struct bus_type *bus;
    struct driver_private *priv;
    int error = 0;
    // 增加该bus的kobject引用计数，这里返回的bus就是platform bus
    bus = bus_get(drv->bus);
    if (!bus)
        return -EINVAL;

    pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);
    // device_drvier的私有数据
    priv = kzalloc(sizeof(*priv), GFP_KERNEL);
    if (!priv) {
        error = -ENOMEM;
        goto out_put_bus;
    }
    klist_init(&priv->klist_devices, NULL, NULL);
    priv->driver = drv;
    drv->p = priv;
    // drivers_kset 本bus下所有的device_driver kobject集合
    priv->kobj.kset = bus->p->drivers_kset;
    // 初始化私有数据kobject
    error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
                        "%s", drv->name);
    if (error)
        goto out_unregister;
    // 将该driver链接到bus的klist_driver上
    klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
    // 进入到autoprobe里面
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

继续来看driver_attach
```c
// drivers/base/dd.c
int driver_attach(struct device_driver *drv)
{
    // 遍历bus上所有的device
    return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}

// drivers/base/dd.c
static int __driver_attach(struct device *dev, void *data)
{
	struct device_driver *drv = data;
	int ret;

	/*
	 * Lock device and try to bind to it. We drop the error
	 * here and always return 0, because we need to keep trying
	 * to bind to devices and some drivers will return an error
	 * simply if it didn't support the device.
	 *
	 * driver_probe_device() will spit a warning if there
	 * is an error.
	 */
    // 这里会调用platform_match(drv, dev)
    // 不匹配的时候会返回0
	ret = driver_match_device(drv, dev);
	if (ret == 0) {
		/* no match */
		return 0;
	} else if (ret == -EPROBE_DEFER) {
		dev_dbg(dev, "Device match requests probe deferral\n");
		driver_deferred_probe_add(dev);
	} else if (ret < 0) {
		dev_dbg(dev, "Bus failed to match device: %d\n", ret);
		return ret;
	} /* ret > 0 means positive match */

	if (driver_allows_async_probing(drv)) {
		/*
		 * Instead of probing the device synchronously we will
		 * probe it asynchronously to allow for more parallelism.
		 *
		 * We only take the device lock here in order to guarantee
		 * that the dev->driver and async_driver fields are protected
		 */
		dev_dbg(dev, "probing driver %s asynchronously\n", drv->name);
		device_lock(dev);
		if (!dev->driver) {
			get_device(dev);
			dev->p->async_driver = drv;
			async_schedule_dev(__driver_attach_async_helper, dev);
		}
		device_unlock(dev);
		return 0;
	}
    // 进行驱动的初始化工作，即调用probe
	device_driver_attach(drv, dev);

	return 0;
}
```

driver_match_device中会调用platform_match来进行驱动和设备的匹配
```c
// drivers/base/base.h
static inline int driver_match_device(struct device_driver *drv,
                                struct device &dev)
{
    return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}

// drivers/base/platform.c
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
设备和驱动匹配顺序：
1. 先用设备树中的compatible属性和platform_driver中的driver中的of_match_table来匹配
2. 再用platform_driver的id_table中的name和platform_device中的name来匹配
3. 最后用platform_device中的name和platform_driver中的driver中的name来匹配


接着来看device_driver_attach()
```c
// drivers/base/dd.c
int device_driver_attach(struct device_driver *drv, struct device *dev)
{
	int ret = 0;
    // 给当前device上锁
	__device_driver_lock(dev, dev->parent);

	/*
	 * If device has been removed or someone has already successfully
	 * bound a driver before us just skip the driver probe call.
	 */
    // 为device绑定驱动
    // dead标志该device已经从系统移除
	if (!dev->p->dead && !dev->driver)
		ret = driver_probe_device(drv, dev);

	__device_driver_unlock(dev, dev->parent);

	return ret;
}
```

再继续来看driver_probe_device()
```c
// drivers/base/dd.c
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
	int ret = 0;

    // 通过该device的kobject中的state_in_sysfs这个标志判断该设备是否已经注册
	if (!device_is_registered(dev))
		return -ENODEV;

	pr_debug("bus: '%s': %s: matched device %s with driver %s\n",
		 drv->bus->name, __func__, dev_name(dev), drv->name);

	pm_runtime_get_suppliers(dev);
	if (dev->parent)
		pm_runtime_get_sync(dev->parent);

	pm_runtime_barrier(dev);
	if (initcall_debug)
		ret = really_probe_debug(dev, drv);
	else
		ret = really_probe(dev, drv);
	pm_request_idle(dev);

	if (dev->parent)
		pm_runtime_put(dev->parent);

	pm_runtime_put_suppliers(dev);
	return ret;
}
```

继续看really_probe()
```c
static int really_probe(struct device *dev, struct device_driver *drv)
{
	int ret = -EPROBE_DEFER;
	int local_trigger_count = atomic_read(&deferred_trigger_count);
	bool test_remove = IS_ENABLED(CONFIG_DEBUG_TEST_DRIVER_REMOVE) &&
			   !drv->suppress_bind_attrs;

    // 如果正处于S3或者S4状态，需要延迟进行驱动初始化
	if (defer_all_probes) {
		/*
		 * Value of defer_all_probes can be set only by
		 * device_block_probing() which, in turn, will call
		 * wait_for_device_probe() right after that to avoid any races.
		 */
		dev_dbg(dev, "Driver %s force probe deferral\n", drv->name);
		driver_deferred_probe_add(dev);
		return ret;
	}

	ret = device_links_check_suppliers(dev);
	if (ret == -EPROBE_DEFER)
		driver_deferred_probe_add_trigger(dev, local_trigger_count);
	if (ret)
		return ret;

	atomic_inc(&probe_count);
	pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
		 drv->bus->name, __func__, drv->name, dev_name(dev));
	if (!list_empty(&dev->devres_head)) {
		dev_crit(dev, "Resources present before probing\n");
		ret = -EBUSY;
		goto done;
	}

re_probe:
	dev->driver = drv;

	/* If using pinctrl, bind pins now before probing */
	ret = pinctrl_bind_pins(dev);
	if (ret)
		goto pinctrl_bind_failed;

	if (dev->bus->dma_configure) {
		ret = dev->bus->dma_configure(dev);
		if (ret)
			goto probe_failed;
	}

	if (driver_sysfs_add(dev)) {
		pr_err("%s: driver_sysfs_add(%s) failed\n",
		       __func__, dev_name(dev));
		goto probe_failed;
	}

	if (dev->pm_domain && dev->pm_domain->activate) {
		ret = dev->pm_domain->activate(dev);
		if (ret)
			goto probe_failed;
	}
    // 这里调用platform_drv_probe()
    // 在platform_drv_probe()里面会调用driver->probe()
    // 这里即phytium_i2c_plat_probe()
	if (dev->bus->probe) {
		ret = dev->bus->probe(dev);
		if (ret)
			goto probe_failed;
	} else if (drv->probe) {
		ret = drv->probe(dev);
		if (ret)
			goto probe_failed;
	}

	if (device_add_groups(dev, drv->dev_groups)) {
		dev_err(dev, "device_add_groups() failed\n");
		goto dev_groups_failed;
	}

	if (dev_has_sync_state(dev) &&
	    device_create_file(dev, &dev_attr_state_synced)) {
		dev_err(dev, "state_synced sysfs add failed\n");
		goto dev_sysfs_state_synced_failed;
	}

	if (test_remove) {
		test_remove = false;

		device_remove_file(dev, &dev_attr_state_synced);
		device_remove_groups(dev, drv->dev_groups);

		if (dev->bus->remove)
			dev->bus->remove(dev);
		else if (drv->remove)
			drv->remove(dev);

		devres_release_all(dev);
		driver_sysfs_remove(dev);
		dev->driver = NULL;
		dev_set_drvdata(dev, NULL);
		if (dev->pm_domain && dev->pm_domain->dismiss)
			dev->pm_domain->dismiss(dev);
		pm_runtime_reinit(dev);

		goto re_probe;
	}

	pinctrl_init_done(dev);

	if (dev->pm_domain && dev->pm_domain->sync)
		dev->pm_domain->sync(dev);

	driver_bound(dev);
	ret = 1;
	pr_debug("bus: '%s': %s: bound device %s to driver %s\n",
		 drv->bus->name, __func__, dev_name(dev), drv->name);
	goto done;

dev_sysfs_state_synced_failed:
	device_remove_groups(dev, drv->dev_groups);
dev_groups_failed:
	if (dev->bus->remove)
		dev->bus->remove(dev);
	else if (drv->remove)
		drv->remove(dev);
probe_failed:
	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_DRIVER_NOT_BOUND, dev);
pinctrl_bind_failed:
	device_links_no_driver(dev);
	devres_release_all(dev);
	arch_teardown_dma_ops(dev);
	driver_sysfs_remove(dev);
	dev->driver = NULL;
	dev_set_drvdata(dev, NULL);
	if (dev->pm_domain && dev->pm_domain->dismiss)
		dev->pm_domain->dismiss(dev);
	pm_runtime_reinit(dev);
	dev_pm_set_driver_flags(dev, 0);

	switch (ret) {
	case -EPROBE_DEFER:
		/* Driver requested deferred probing */
		dev_dbg(dev, "Driver %s requests probe deferral\n", drv->name);
		driver_deferred_probe_add_trigger(dev, local_trigger_count);
		break;
	case -ENODEV:
	case -ENXIO:
		pr_debug("%s: probe of %s rejects match %d\n",
			 drv->name, dev_name(dev), ret);
		break;
	default:
		/* driver matched but the probe failed */
		pr_warn("%s: probe of %s failed with error %d\n",
			drv->name, dev_name(dev), ret);
	}
	/*
	 * Ignore errors returned by ->probe so that the next driver can try
	 * its luck.
	 */
	ret = 0;
done:
	atomic_dec(&probe_count);
	wake_up_all(&probe_waitqueue);
	return ret;
}
```




