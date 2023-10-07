---
title: Linux电源管理框架
tags:
  - Linux
  - Power Management Framework
categories: Linux
abbrlink: '43564739'
date: 2023-01-13 08:12:46
---

### 概述
Linux内核中电源管理主要涉及到供电、充电、时钟、频率、电压、睡眠唤醒等内容

<!-- more -->

Linux内核电源管理的组成：
1. Gerneric PM：传统意义上的电源管理，主要是Power off, sleep, hibernate, restart等，涉及到各个层级的suspend, resume, prepare_suspend, suspend_early, suspend_noirq, resume_late, complete
2. Runtime PM和wakelock，kernel层级的调度
3. CPU Idle
4. Clock Framework，时钟资源
5. CPU Freq, Device Freq，CPU和设备频率相关
6. OPP，芯片和设备正常工作的电压和频率组合
7. PM QOS，系统运行状态的质量
8. Regulator Framework，电压和电流
9. Power Supply，电源供电状态

### Linux内核电源状态
在Linux内核中，将电源划分为如下几个状态
| ACPI | State | Linux State Drscription |
| :-: | :-: | :- |
| S0 | On | Working |
| S1 | Standby | CPU and RAM are powered but not executed |
| S2 | N/A | N/A |
| S3 | Suspend to RAM | CPU is off, RAM is powered and the running content is saved to RAM |
| S4 | Suspend to Disk | All content is saved to Disk and power down |
| S5 | Shutdown | Shutdown the system |

### Generic PM
Generic PM指的是Linux系统中常规的电源管理，包括关机、待机、重启等

Generic PM的软件层次：
+ API Layer：用于向用户空间提供接口，其中关机、重启的接口形式是系统调用，Hibernate、Suspend的接口形式是sysfs
+ PM Core：位于`kernel/power`目录下，主要处理和硬件无关的核心逻辑
+ PM Driver：分为两个部分，一个是和体系结构无关的Driver，提供pm框架；另一部分是具体的体系结构相关的driver，也是开发者关注的部分

#### Generic PM reboot的过程

kernel支持的reboot的方式定义在`include/uapi/linux/reboot.h`中，代码如下：
```c
// include/uapi/linux/reboot.h
/*
 * Magic values required to use _reboot() system call.
 */

#define	LINUX_REBOOT_MAGIC1	0xfee1dead
#define	LINUX_REBOOT_MAGIC2	672274793
#define	LINUX_REBOOT_MAGIC2A	85072278
#define	LINUX_REBOOT_MAGIC2B	369367448
#define	LINUX_REBOOT_MAGIC2C	537993216


/*
 * Commands accepted by the _reboot() system call.
 *
 * RESTART     Restart system using default command and mode.
 * HALT        Stop OS and give system control to ROM monitor, if any.
 * CAD_ON      Ctrl-Alt-Del sequence causes RESTART command.
 * CAD_OFF     Ctrl-Alt-Del sequence sends SIGINT to init task.
 * POWER_OFF   Stop OS and remove all power from system, if possible.
 * RESTART2    Restart system using given command string.
 * SW_SUSPEND  Suspend system using software suspend if compiled in.
 * KEXEC       Restart system using a previously loaded Linux kernel
 */

#define	LINUX_REBOOT_CMD_RESTART	0x01234567
#define	LINUX_REBOOT_CMD_HALT		0xCDEF0123
#define	LINUX_REBOOT_CMD_CAD_ON		0x89ABCDEF
#define	LINUX_REBOOT_CMD_CAD_OFF	0x00000000
#define	LINUX_REBOOT_CMD_POWER_OFF	0x4321FEDC
#define	LINUX_REBOOT_CMD_RESTART2	0xA1B2C3D4
#define	LINUX_REBOOT_CMD_SW_SUSPEND	0xD000FCE2
#define	LINUX_REBOOT_CMD_KEXEC		0x45584543
```

#### reboot过程的代码分析
reboot的系统调用定义在`kernel/reboot.c`中，其代码如下
```c
/*
 * Reboot system call: for obvious reasons only root may call it,
 * and even root needs to set up some magic numbers in the registers
 * so that some mistake won't make this reboot the whole machine.
 * You can also set the meaning of the ctrl-alt-del-key here.
 *
 * reboot doesn't sync: do that yourself before calling this.
 */
SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,
		void __user *, arg)
{
	struct pid_namespace *pid_ns = task_active_pid_ns(current);
	char buffer[256];
	int ret = 0;

	/* We only trust the superuser with rebooting the system. */
	// 判断调用者的用户权限，只有root用户才能执行
	// 从current中取到pid_namespace，再通过pid_namespace中的userspace判断权限
	if (!ns_capable(pid_ns->user_ns, CAP_SYS_BOOT))
		return -EPERM;

	/* For safety, we require "magic" arguments. */
	// 判断magic number是否匹配
	if (magic1 != LINUX_REBOOT_MAGIC1 ||
			(magic2 != LINUX_REBOOT_MAGIC2 &&
			magic2 != LINUX_REBOOT_MAGIC2A &&
			magic2 != LINUX_REBOOT_MAGIC2B &&
			magic2 != LINUX_REBOOT_MAGIC2C))
		return -EINVAL;

	/*
	 * If pid namespaces are enabled and the current task is in a child
	 * pid_namespace, the command is handled by reboot_pid_ns() which will
	 * call do_exit().
	 */
	ret = reboot_pid_ns(pid_ns, cmd);
	if (ret)
		return ret;

	/* Instead of trying to make the power_off code look like
	 * halt when pm_power_off is not set do it the easy way.
	 */
	if ((cmd == LINUX_REBOOT_CMD_POWER_OFF) && !kernel_can_power_off())
		cmd = LINUX_REBOOT_CMD_HALT;

	mutex_lock(&system_transition_mutex);
	switch (cmd) {
	case LINUX_REBOOT_CMD_RESTART:
		kernel_restart(NULL);
		break;

	case LINUX_REBOOT_CMD_CAD_ON:
		C_A_D = 1;
		break;

	case LINUX_REBOOT_CMD_CAD_OFF:
		C_A_D = 0;
		break;

	case LINUX_REBOOT_CMD_HALT:
		kernel_halt();
		do_exit(0);

	case LINUX_REBOOT_CMD_POWER_OFF:
		kernel_power_off();
		do_exit(0);
		break;

	case LINUX_REBOOT_CMD_RESTART2:
		ret = strncpy_from_user(&buffer[0], arg, sizeof(buffer) - 1);
		if (ret < 0) {
			ret = -EFAULT;
			break;
		}
		buffer[sizeof(buffer) - 1] = '\0';

		kernel_restart(buffer);
		break;

#ifdef CONFIG_KEXEC_CORE
	case LINUX_REBOOT_CMD_KEXEC:
		ret = kernel_kexec();
		break;
#endif

#ifdef CONFIG_HIBERNATION
	case LINUX_REBOOT_CMD_SW_SUSPEND:
		ret = hibernate();
		break;
#endif

	default:
		ret = -EINVAL;
		break;
	}
	mutex_unlock(&system_transition_mutex);
	return ret;
```

`kernel_restart()`位于`kernel/reboot.c`中，代码如下：
```c
// kernel/reboot.c
/**
 *	kernel_restart - reboot the system
 *	@cmd: pointer to buffer containing command to execute for restart
 *		or %NULL
 *
 *	Shutdown everything and perform a clean reboot.
 *	This is not safe to call in interrupt context.
 */
void kernel_restart(char *cmd)
{
	kernel_restart_prepare(cmd);
	do_kernel_restart_prepare();
	migrate_to_reboot_cpu();
	syscore_shutdown();
	if (!cmd)
		pr_emerg("Restarting system\n");
	else
		pr_emerg("Restarting system with command '%s'\n", cmd);
	kmsg_dump(KMSG_DUMP_SHUTDOWN);
	machine_restart(cmd);
}
EXPORT_SYMBOL_GPL(kernel_restart);

// include/linux/kernel.h
// 系统状态枚举变量
extern enum system_states {
	SYSTEM_BOOTING,
	SYSTEM_SCHEDULING,
	SYSTEM_FREEING_INITMEM,
	SYSTEM_RUNNING,
	SYSTEM_HALT,
	SYSTEM_POWER_OFF,
	SYSTEM_RESTART,
	SYSTEM_SUSPEND,
} system_state;


// kernel/reboot.c
void kernel_restart_prepare(char *cmd)
{
	// reboot_notifier_list定义在kernel/notifier.c中
	// 向该通知链上发送SYS_RESTART事件，notifier执行各自的回调函数
	blocking_notifier_call_chain(&reboot_notifier_list, SYS_RESTART, cmd);
	// system_state 定义在init/main.c中
	// 更新system_state的值
	system_state = SYSTEM_RESTART;
	usermodehelper_disable();
	// 调用device_shutdown，关闭所有设备
	device_shutdown();
}
```

来看一下`device_shutdown()`的实现

`device_shutdown()`中和设备模型相关的逻辑：
+ 每个设备`struct device`都会保存该设备的驱动指针`struct device_driver`，以及该设备所在总线`struct bus_type`的指针
+ 设备驱动中有一个名称为`shutdown()`的回调函数，用于在`device_shutdown`时，关闭该设备
+ 总线中也有一个名称为`shutdown()`的回调函数，用于在`device_shutdown`时，关闭该设备
+ 系统中所有的设备，都存在于`/sys/devices/`目录下面，该目录由名称`device_kset`的kset表示，kset中会使用一个链表保存其下所有的kobject。以`device_kset`为root节点，将内核中所有的设备，组织成一个树状结构

```c
// drivers/base/core.c
/* /sys/devices */
struct kset *devices_kset;

/**
 * device_shutdown - call ->shutdown() on each device to shutdown.
 */
void device_shutdown(void)
{
	struct device *dev, *parent;

	wait_for_device_probe();
	device_block_probing();
	// cpufreq_suspended标志置为true
	cpufreq_suspend();

	spin_lock(&devices_kset->list_lock);
	/*
	 * Walk the devices list backward, shutting down each in turn.
	 * Beware that device unplug events may also start pulling
	 * devices offline, even as the system is shutting down.
	 */
	while (!list_empty(&devices_kset->list)) {
		dev = list_entry(devices_kset->list.prev, struct device,
				kobj.entry);

		/*
		 * hold reference count of device's parent to
		 * prevent it from being freed because parent's
		 * lock is to be held
		 */
		parent = get_device(dev->parent);
		get_device(dev);
		/*
		 * Make sure the device is off the kset list, in the
		 * event that dev->*->shutdown() doesn't remove it.
		 */
		// 将该device从devices_kset上删除
		list_del_init(&dev->kobj.entry);
		spin_unlock(&devices_kset->list_lock);

		/* hold lock to avoid race with probe/release */
		if (parent)
			device_lock(parent);
		device_lock(dev);

		/* Don't allow any more runtime suspends */
		// 直接增加dev->power.usage_count的计数，不做其他处理
		// 这样不让设备进入到runtime suspend
		pm_runtime_get_noresume(dev);
		// 查看设备当前的状态
		// 1. 若现在正有唤醒的请求则唤醒设备
		// 2. 取消当前挂入到workqueue的工作任务
		// 3. 若设备正处于RPM_SUSPENDING或RPM_RESUMING则等待work执行完
		pm_runtime_barrier(dev);
		// 调用shutdown接口
		// 调用顺序:
		// class->shutdown_pre()
		// bus->shutdown()
		// driver->shutdown()
		if (dev->class && dev->class->shutdown_pre) {
			if (initcall_debug)
				dev_info(dev, "shutdown_pre\n");
			dev->class->shutdown_pre(dev);
		}
		if (dev->bus && dev->bus->shutdown) {
			if (initcall_debug)
				dev_info(dev, "shutdown\n");
			dev->bus->shutdown(dev);
		} else if (dev->driver && dev->driver->shutdown) {
			if (initcall_debug)
				dev_info(dev, "shutdown\n");
			dev->driver->shutdown(dev);
		}

		device_unlock(dev);
		if (parent)
			device_unlock(parent);
		// 引用计数减1
		put_device(dev);
		put_device(parent);

		spin_lock(&devices_kset->list_lock);
	}
	spin_unlock(&devices_kset->list_lock);
}
```

### Runtime PM
runtime PM的思想：每个设备都处理好自己的电源管理工作，尽量以最低的能耗完成交代的任务，尽量在不需要工作的时候进入低功耗状态，尽量不和其它模块有过多的耦合。

`device_driver`需要提供3个回调函数`runtime_suspend`, `runtime_resume`, `runtime_idle`，分别用于suspend device, resume device和idle device，它们一般由runtime pm core在合适的时机进行调用，以便device节能

`device driver`会在适当的时机调用runtime pm core提供put, get系列接口，汇报device当前的状态，runtime pm core会为每个device维护一个引用计数`device->power.usage_count`，get时增加引用计数，put时减少引用计数，当计数为0时，表明device不再被使用，可以立即或一段时间后suspend

```c
// device_driver需要实现的runtime接口
struct dev_pm_ops {
	...
	int (*runtime_suspend)(struct device *dev);
	int (*runtime_resume)(struct device *dev);
	int (*runtime_idle)(struct device *dev);
};

struct device {
	...
	struct dev_pm_info power;
	...
};

// device中的电源状态信息
struct dev_pm_info {
	pm_message_t		power_state;
	unsigned int		can_wakeup:1;
	unsigned int		async_suspend:1;
	bool			in_dpm_list:1;	/* Owned by the PM core */
	bool			is_prepared:1;	/* Owned by the PM core */
	bool			is_suspended:1;	/* Ditto */
	bool			is_noirq_suspended:1;
	bool			is_late_suspended:1;
	bool			no_pm:1;
	bool			early_init:1;	/* Owned by the PM core */
	bool			direct_complete:1;	/* Owned by the PM core */
	u32			driver_flags;
	spinlock_t		lock;
#ifdef CONFIG_PM_SLEEP
	struct list_head	entry;
	struct completion	completion;
	struct wakeup_source	*wakeup;
	bool			wakeup_path:1;
	bool			syscore:1;
	bool			no_pm_callbacks:1;	/* Owned by the PM core */
	unsigned int		must_resume:1;	/* Owned by the PM core */
	unsigned int		may_skip_resume:1;	/* Set by subsystems */
#else
	unsigned int		should_wakeup:1;
#endif
#ifdef CONFIG_PM
	struct hrtimer		suspend_timer;
	u64			timer_expires;
	struct work_struct	work;
	wait_queue_head_t	wait_queue;
	struct wake_irq		*wakeirq;
	atomic_t		usage_count;
	atomic_t		child_count;
	unsigned int		disable_depth:3;
	unsigned int		idle_notification:1;
	unsigned int		request_pending:1;
	unsigned int		deferred_resume:1;
	unsigned int		needs_force_resume:1;
	unsigned int		runtime_auto:1;
	bool			ignore_children:1;
	unsigned int		no_callbacks:1;
	unsigned int		irq_safe:1;
	unsigned int		use_autosuspend:1;
	unsigned int		timer_autosuspends:1;
	unsigned int		memalloc_noio:1;
	unsigned int		links_count;
	enum rpm_request	request;
	enum rpm_status		runtime_status;
	enum rpm_status		last_status;
	int			runtime_error;
	int			autosuspend_delay;
	u64			last_busy;
	u64			active_time;
	u64			suspended_time;
	u64			accounting_timestamp;
#endif
	struct pm_subsys_data	*subsys_data;  /* Owned by the subsystem. */
	void (*set_latency_tolerance)(struct device *, s32);
	struct dev_pm_qos	*qos;
};
```

在runtime PM的过程中，设备可处于如下四种状态
+ RPM_ACTIVE：设备处于正常工作的状态
+ RPM_SUSPENDED：设备处于suspend状态
+ RPM_RESUMING：设备runtime_resume正在被执行
+ RPM_SUSPENDING：设备runtime_suspend正在被执行

#### runtime PM的异步和同步
设备驱动代码可在进程和中断两种上下文执行，因此put和get等接口，要么是由用户进程调用，要么是由中断处理函数调用。由于这些接口可能会执行device的.runtime_xxx回调函数，而这些接口的执行时间是不确定的，有些可能还会睡眠等待。

runtime PM core提供默认的接口(pm_runtime_get, pm_runtime_put)，这两个接口采用异步调用的方式（传入ASYNC标志），会启动一个work挂入到workqueue，在worker线程中调用runtime_suspend, runtime_resume回调函数，这可以保证设备驱动之外的其它模块正常运行

如果驱动工程师很清楚的知道自己要做什么，也可以使用同步接口（pm_runtime_get_sync, pm_runtime_put_sync），它们会直接调用runtime_suspend, runtime_resume回调函数

#### runtime PM API接口
runtime PM提供的接口位于`include/linux/pm_runtime.h`

1. 



### power supply子系统
#### power supply软件架构
Linux内核抽象出来power supply子系统为驱动提供了统一的框架，功能包括：
1. 抽象power supply设备的共性，向用户空间提供统一的API
2. 为底层power supply驱动的编写，提供简单统一的方式，同时封装并实现公共逻辑

power supply class位于`driver/power/supply`目录中，主要由3部分组成：
1. power supply core：用于抽象核心数据结构，实现公共逻辑
2. power supply sysfs：实现sysfs以及uevent功能
3. power supply leds：给予Linux led class，提供power supply设备状态指示的通用实现

驱动工程师可以基于power supply class实现具体的power supply driver，主要处理平台相关、硬件相关的逻辑

相关数据结构
```c

/* Description of power supply */
// include/linux/power_supply.h
// power supply设备描述
struct power_supply_desc {
    // power supply设备名称
	const char *name;
    // power supply设备的类型，电池 UPS 充电器等
	enum power_supply_type type;
	const enum power_supply_usb_type *usb_types;
	size_t num_usb_types;
    // power supply设备属性列表
	const enum power_supply_property *properties;
    // 属性个数
	size_t num_properties;

	/*
	 * Functions for drivers implementing power supply class.
	 * These shouldn't be called directly by other drivers for accessing
	 * this power supply. Instead use power_supply_*() functions (for
	 * example power_supply_get_property()).
	 */
    // power supply驱动需要实现的接口
	int (*get_property)(struct power_supply *psy,
			    enum power_supply_property psp,
			    union power_supply_propval *val);
	int (*set_property)(struct power_supply *psy,
			    enum power_supply_property psp,
			    const union power_supply_propval *val);
	/*
	 * property_is_writeable() will be called during registration
	 * of power supply. If this happens during device probe then it must
	 * not access internal data of device (because probe did not end).
	 */
    // 返回指定的属性值是否可写
	int (*property_is_writeable)(struct power_supply *psy,
				     enum power_supply_property psp);
	void (*external_power_changed)(struct power_supply *psy);
	void (*set_charged)(struct power_supply *psy);

	/*
	 * Set if thermal zone should not be created for this power supply.
	 * For example for virtual supplies forwarding calls to actual
	 * sensors or other supplies.
	 */
	bool no_thermal;
	/* For APM emulation, think legacy userspace. */
	int use_for_apm;
};


// include/linux/power_supply.h
struct power_supply {
    // power supply设备描述
	const struct power_supply_desc *desc;

	char **supplied_to;
	size_t num_supplicants;

	char **supplied_from;
	size_t num_supplies;
	struct device_node *of_node;

	/* Driver private data */
	void *drv_data;

	/* private */
	struct device dev;
    // 用户处理状态改变的work queue, 当power supply设备的状态发生改变，启动一个
    // workqueue，查询并通知所有的supplicants
	struct work_struct changed_work;
	struct delayed_work deferred_register_work;
	spinlock_t changed_lock;
	bool changed;
	bool initialized;
	bool removing;
	atomic_t use_cnt;
#ifdef CONFIG_THERMAL
	struct thermal_zone_device *tzd;
	struct thermal_cooling_device *tcd;
#endif

    // power supply设备LED状态指示
#ifdef CONFIG_LEDS_TRIGGERS
	struct led_trigger *charging_full_trig;
	char *charging_full_trig_name;
	struct led_trigger *charging_trig;
	char *charging_trig_name;
	struct led_trigger *full_trig;
	char *full_trig_name;
	struct led_trigger *online_trig;
	char *online_trig_name;
	struct led_trigger *charging_blink_full_solid_trig;
	char *charging_blink_full_solid_trig_name;
#endif
};
```

power supply设备的类型由`enum power_supply_type`定义，如下：
```c
enum power_supply_type {
    // 未知设备
	POWER_SUPPLY_TYPE_UNKNOWN = 0,
    // 电池供电
	POWER_SUPPLY_TYPE_BATTERY,
    // 不间断供电设备(Uninterruptible Power Supply)
	POWER_SUPPLY_TYPE_UPS,
    // 主供电设备，例如笔记本电源适配器
	POWER_SUPPLY_TYPE_MAINS,
	POWER_SUPPLY_TYPE_USB,			/* Standard Downstream Port */
	POWER_SUPPLY_TYPE_USB_DCP,		/* Dedicated Charging Port */
	POWER_SUPPLY_TYPE_USB_CDP,		/* Charging Downstream Port */
	POWER_SUPPLY_TYPE_USB_ACA,		/* Accessory Charger Adapters */
	POWER_SUPPLY_TYPE_USB_TYPE_C,		/* Type C Port */
	POWER_SUPPLY_TYPE_USB_PD,		/* Power Delivery Port */
	POWER_SUPPLY_TYPE_USB_PD_DRP,		/* PD Dual Role Port */
	POWER_SUPPLY_TYPE_APPLE_BRICK_ID,	/* Apple Charging Method */
	POWER_SUPPLY_TYPE_WIRELESS,		/* Wireless */
};
```

power supply设备属性由`enum power_supply_property`定义，如下：
```c
enum power_supply_property {
	/* Properties of type `int' */
	POWER_SUPPLY_PROP_STATUS = 0,
	POWER_SUPPLY_PROP_CHARGE_TYPE,
	POWER_SUPPLY_PROP_HEALTH,
	POWER_SUPPLY_PROP_PRESENT,
	POWER_SUPPLY_PROP_ONLINE,
	POWER_SUPPLY_PROP_AUTHENTIC,
	POWER_SUPPLY_PROP_TECHNOLOGY,
	POWER_SUPPLY_PROP_CYCLE_COUNT,
	POWER_SUPPLY_PROP_VOLTAGE_MAX,
	POWER_SUPPLY_PROP_VOLTAGE_MIN,
	POWER_SUPPLY_PROP_VOLTAGE_MAX_DESIGN,
	POWER_SUPPLY_PROP_VOLTAGE_MIN_DESIGN,
	POWER_SUPPLY_PROP_VOLTAGE_NOW,
	POWER_SUPPLY_PROP_VOLTAGE_AVG,
	POWER_SUPPLY_PROP_VOLTAGE_OCV,
	POWER_SUPPLY_PROP_VOLTAGE_BOOT,
	POWER_SUPPLY_PROP_CURRENT_MAX,
	POWER_SUPPLY_PROP_CURRENT_NOW,
	POWER_SUPPLY_PROP_CURRENT_AVG,
	POWER_SUPPLY_PROP_CURRENT_BOOT,
	POWER_SUPPLY_PROP_POWER_NOW,
	POWER_SUPPLY_PROP_POWER_AVG,
	POWER_SUPPLY_PROP_CHARGE_FULL_DESIGN,
	POWER_SUPPLY_PROP_CHARGE_EMPTY_DESIGN,
	POWER_SUPPLY_PROP_CHARGE_FULL,
	POWER_SUPPLY_PROP_CHARGE_EMPTY,
	POWER_SUPPLY_PROP_CHARGE_NOW,
	POWER_SUPPLY_PROP_CHARGE_AVG,
	POWER_SUPPLY_PROP_CHARGE_COUNTER,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_CURRENT,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_CURRENT_MAX,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_VOLTAGE,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_VOLTAGE_MAX,
	POWER_SUPPLY_PROP_CHARGE_CONTROL_LIMIT,
	POWER_SUPPLY_PROP_CHARGE_CONTROL_LIMIT_MAX,
	POWER_SUPPLY_PROP_CHARGE_CONTROL_START_THRESHOLD, /* in percents! */
	POWER_SUPPLY_PROP_CHARGE_CONTROL_END_THRESHOLD, /* in percents! */
	POWER_SUPPLY_PROP_INPUT_CURRENT_LIMIT,
	POWER_SUPPLY_PROP_INPUT_VOLTAGE_LIMIT,
	POWER_SUPPLY_PROP_INPUT_POWER_LIMIT,
	POWER_SUPPLY_PROP_ENERGY_FULL_DESIGN,
	POWER_SUPPLY_PROP_ENERGY_EMPTY_DESIGN,
	POWER_SUPPLY_PROP_ENERGY_FULL,
	POWER_SUPPLY_PROP_ENERGY_EMPTY,
	POWER_SUPPLY_PROP_ENERGY_NOW,
	POWER_SUPPLY_PROP_ENERGY_AVG,
	POWER_SUPPLY_PROP_CAPACITY, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_ALERT_MIN, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_ALERT_MAX, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_ERROR_MARGIN, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_LEVEL,
	POWER_SUPPLY_PROP_TEMP,
	POWER_SUPPLY_PROP_TEMP_MAX,
	POWER_SUPPLY_PROP_TEMP_MIN,
	POWER_SUPPLY_PROP_TEMP_ALERT_MIN,
	POWER_SUPPLY_PROP_TEMP_ALERT_MAX,
	POWER_SUPPLY_PROP_TEMP_AMBIENT,
	POWER_SUPPLY_PROP_TEMP_AMBIENT_ALERT_MIN,
	POWER_SUPPLY_PROP_TEMP_AMBIENT_ALERT_MAX,
	POWER_SUPPLY_PROP_TIME_TO_EMPTY_NOW,
	POWER_SUPPLY_PROP_TIME_TO_EMPTY_AVG,
	POWER_SUPPLY_PROP_TIME_TO_FULL_NOW,
	POWER_SUPPLY_PROP_TIME_TO_FULL_AVG,
	POWER_SUPPLY_PROP_TYPE, /* use power_supply.type instead */
	POWER_SUPPLY_PROP_USB_TYPE,
	POWER_SUPPLY_PROP_SCOPE,
	POWER_SUPPLY_PROP_PRECHARGE_CURRENT,
	POWER_SUPPLY_PROP_CHARGE_TERM_CURRENT,
	POWER_SUPPLY_PROP_CALIBRATE,
	POWER_SUPPLY_PROP_MANUFACTURE_YEAR,
	POWER_SUPPLY_PROP_MANUFACTURE_MONTH,
	POWER_SUPPLY_PROP_MANUFACTURE_DAY,
	/* Properties of type `const char *' */
	POWER_SUPPLY_PROP_MODEL_NAME,
	POWER_SUPPLY_PROP_MANUFACTURER,
	POWER_SUPPLY_PROP_SERIAL_NUMBER,
};
```

power supply class向power supply驱动提供统一的驱动编程接口
1. 
