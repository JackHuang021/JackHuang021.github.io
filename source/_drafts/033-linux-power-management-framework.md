---
title: linux电源管理框架
tags:
  - linux
  - Power Management Framework
categories: linux
abbrlink: '43564739'
date: 2023-01-13 08:12:46
---

### 概述
linux内核中电源管理主要涉及到供电、充电、时钟、频率、电压、睡眠唤醒等内容

<!-- more -->

linux内核电源管理的组成：
1. Gerneric PM：传统意义上的电源管理，主要是Power off, sleep, hibernate, restart等，涉及到各个层级的suspend, resume, prepare_suspend, suspend_early, suspend_noirq, resume_late, complete
2. Runtime PM和wakelock，kernel层级的调度
3. CPU Idle
4. Clock Framework，时钟资源
5. CPU Freq, Device Freq，CPU和设备频率相关
6. OPP，芯片和设备正常工作的电压和频率组合
7. PM QOS，系统运行状态的质量
8. Regulator Framework，电压和电流
9. Power Supply，电源供电状态

### 设备电源管理接口
linux电源管理中，相当多的部分是在处理suspend，Runtime PM等功能。而这些功能都基于一套相似的逻辑，即power management interface，该interface的代码实现在`include/linux/pm.h`, `drivers/base/power/main.c`等文件中

power management interface的主要功能：
+ 对下，定义device PM相关的回调函数，让各个dirver实现
+ 对上，实现统一的PM操作函数，供PM核心逻辑调用

#### device PM回调函数
在一个系统中，数量最多的是设备，耗电最多的也是设备，设备的电源管理是linux电源管理的核心内容，设备电源管理最核心的操作就是：**在合适的时机，将设备置为合理的状态**，device pm callbacks的目的就是：定义一套统一的方式，让设备在特定的时机进入合适的状态

这些pm callbacks统一封装在一个数据结构`struct dev_pm_ops`，上层的数据结构只需要包含这个结构即可，旧版本的内核在`struct device; struct class; struct bus_type`中也有suspend/resume类型的callbacks，为了兼容旧的设计，这些接口也保留下来了，只是不建议使用

`struct dev_pm_ops`定义在`include/linux/pm.h`中，内核也对各个接口有详细的注释
```c
// include/linux/pm.h
/**
 * struct dev_pm_ops - device PM callbacks.
 *
 * @prepare: The principal role of this callback is to prevent new children of
 *	the device from being registered after it has returned (the driver's
 *	subsystem and generally the rest of the kernel is supposed to prevent
 *	new calls to the probe method from being made too once @prepare() has
 *	succeeded).  If @prepare() detects a situation it cannot handle (e.g.
 *	registration of a child already in progress), it may return -EAGAIN, so
 *	that the PM core can execute it once again (e.g. after a new child has
 *	been registered) to recover from the race condition.
 *	This method is executed for all kinds of suspend transitions and is
 *	followed by one of the suspend callbacks: @suspend(), @freeze(), or
 *	@poweroff().  If the transition is a suspend to memory or standby (that
 *	is, not related to hibernation), the return value of @prepare() may be
 *	used to indicate to the PM core to leave the device in runtime suspend
 *	if applicable.  Namely, if @prepare() returns a positive number, the PM
 *	core will understand that as a declaration that the device appears to be
 *	runtime-suspended and it may be left in that state during the entire
 *	transition and during the subsequent resume if all of its descendants
 *	are left in runtime suspend too.  If that happens, @complete() will be
 *	executed directly after @prepare() and it must ensure the proper
 *	functioning of the device after the system resume.
 *	The PM core executes subsystem-level @prepare() for all devices before
 *	starting to invoke suspend callbacks for any of them, so generally
 *	devices may be assumed to be functional or to respond to runtime resume
 *	requests while @prepare() is being executed.  However, device drivers
 *	may NOT assume anything about the availability of user space at that
 *	time and it is NOT valid to request firmware from within @prepare()
 *	(it's too late to do that).  It also is NOT valid to allocate
 *	substantial amounts of memory from @prepare() in the GFP_KERNEL mode.
 *	[To work around these limitations, drivers may register suspend and
 *	hibernation notifiers to be executed before the freezing of tasks.]
 *
 * @complete: Undo the changes made by @prepare().  This method is executed for
 *	all kinds of resume transitions, following one of the resume callbacks:
 *	@resume(), @thaw(), @restore().  Also called if the state transition
 *	fails before the driver's suspend callback: @suspend(), @freeze() or
 *	@poweroff(), can be executed (e.g. if the suspend callback fails for one
 *	of the other devices that the PM core has unsuccessfully attempted to
 *	suspend earlier).
 *	The PM core executes subsystem-level @complete() after it has executed
 *	the appropriate resume callbacks for all devices.  If the corresponding
 *	@prepare() at the beginning of the suspend transition returned a
 *	positive number and the device was left in runtime suspend (without
 *	executing any suspend and resume callbacks for it), @complete() will be
 *	the only callback executed for the device during resume.  In that case,
 *	@complete() must be prepared to do whatever is necessary to ensure the
 *	proper functioning of the device after the system resume.  To this end,
 *	@complete() can check the power.direct_complete flag of the device to
 *	learn whether (unset) or not (set) the previous suspend and resume
 *	callbacks have been executed for it.
 *
 * @suspend: Executed before putting the system into a sleep state in which the
 *	contents of main memory are preserved.  The exact action to perform
 *	depends on the device's subsystem (PM domain, device type, class or bus
 *	type), but generally the device must be quiescent after subsystem-level
 *	@suspend() has returned, so that it doesn't do any I/O or DMA.
 *	Subsystem-level @suspend() is executed for all devices after invoking
 *	subsystem-level @prepare() for all of them.
 *
 * @suspend_late: Continue operations started by @suspend().  For a number of
 *	devices @suspend_late() may point to the same callback routine as the
 *	runtime suspend callback.
 *
 * @resume: Executed after waking the system up from a sleep state in which the
 *	contents of main memory were preserved.  The exact action to perform
 *	depends on the device's subsystem, but generally the driver is expected
 *	to start working again, responding to hardware events and software
 *	requests (the device itself may be left in a low-power state, waiting
 *	for a runtime resume to occur).  The state of the device at the time its
 *	driver's @resume() callback is run depends on the platform and subsystem
 *	the device belongs to.  On most platforms, there are no restrictions on
 *	availability of resources like clocks during @resume().
 *	Subsystem-level @resume() is executed for all devices after invoking
 *	subsystem-level @resume_noirq() for all of them.
 *
 * @resume_early: Prepare to execute @resume().  For a number of devices
 *	@resume_early() may point to the same callback routine as the runtime
 *	resume callback.
 *
 * @freeze: Hibernation-specific, executed before creating a hibernation image.
 *	Analogous to @suspend(), but it should not enable the device to signal
 *	wakeup events or change its power state.  The majority of subsystems
 *	(with the notable exception of the PCI bus type) expect the driver-level
 *	@freeze() to save the device settings in memory to be used by @restore()
 *	during the subsequent resume from hibernation.
 *	Subsystem-level @freeze() is executed for all devices after invoking
 *	subsystem-level @prepare() for all of them.
 *
 * @freeze_late: Continue operations started by @freeze().  Analogous to
 *	@suspend_late(), but it should not enable the device to signal wakeup
 *	events or change its power state.
 *
 * @thaw: Hibernation-specific, executed after creating a hibernation image OR
 *	if the creation of an image has failed.  Also executed after a failing
 *	attempt to restore the contents of main memory from such an image.
 *	Undo the changes made by the preceding @freeze(), so the device can be
 *	operated in the same way as immediately before the call to @freeze().
 *	Subsystem-level @thaw() is executed for all devices after invoking
 *	subsystem-level @thaw_noirq() for all of them.  It also may be executed
 *	directly after @freeze() in case of a transition error.
 *
 * @thaw_early: Prepare to execute @thaw().  Undo the changes made by the
 *	preceding @freeze_late().
 *
 * @poweroff: Hibernation-specific, executed after saving a hibernation image.
 *	Analogous to @suspend(), but it need not save the device's settings in
 *	memory.
 *	Subsystem-level @poweroff() is executed for all devices after invoking
 *	subsystem-level @prepare() for all of them.
 *
 * @poweroff_late: Continue operations started by @poweroff().  Analogous to
 *	@suspend_late(), but it need not save the device's settings in memory.
 *
 * @restore: Hibernation-specific, executed after restoring the contents of main
 *	memory from a hibernation image, analogous to @resume().
 *
 * @restore_early: Prepare to execute @restore(), analogous to @resume_early().
 *
 * @suspend_noirq: Complete the actions started by @suspend().  Carry out any
 *	additional operations required for suspending the device that might be
 *	racing with its driver's interrupt handler, which is guaranteed not to
 *	run while @suspend_noirq() is being executed.
 *	It generally is expected that the device will be in a low-power state
 *	(appropriate for the target system sleep state) after subsystem-level
 *	@suspend_noirq() has returned successfully.  If the device can generate
 *	system wakeup signals and is enabled to wake up the system, it should be
 *	configured to do so at that time.  However, depending on the platform
 *	and device's subsystem, @suspend() or @suspend_late() may be allowed to
 *	put the device into the low-power state and configure it to generate
 *	wakeup signals, in which case it generally is not necessary to define
 *	@suspend_noirq().
 *
 * @resume_noirq: Prepare for the execution of @resume() by carrying out any
 *	operations required for resuming the device that might be racing with
 *	its driver's interrupt handler, which is guaranteed not to run while
 *	@resume_noirq() is being executed.
 *
 * @freeze_noirq: Complete the actions started by @freeze().  Carry out any
 *	additional operations required for freezing the device that might be
 *	racing with its driver's interrupt handler, which is guaranteed not to
 *	run while @freeze_noirq() is being executed.
 *	The power state of the device should not be changed by either @freeze(),
 *	or @freeze_late(), or @freeze_noirq() and it should not be configured to
 *	signal system wakeup by any of these callbacks.
 *
 * @thaw_noirq: Prepare for the execution of @thaw() by carrying out any
 *	operations required for thawing the device that might be racing with its
 *	driver's interrupt handler, which is guaranteed not to run while
 *	@thaw_noirq() is being executed.
 *
 * @poweroff_noirq: Complete the actions started by @poweroff().  Analogous to
 *	@suspend_noirq(), but it need not save the device's settings in memory.
 *
 * @restore_noirq: Prepare for the execution of @restore() by carrying out any
 *	operations required for thawing the device that might be racing with its
 *	driver's interrupt handler, which is guaranteed not to run while
 *	@restore_noirq() is being executed.  Analogous to @resume_noirq().
 *
 * @runtime_suspend: Prepare the device for a condition in which it won't be
 *	able to communicate with the CPU(s) and RAM due to power management.
 *	This need not mean that the device should be put into a low-power state.
 *	For example, if the device is behind a link which is about to be turned
 *	off, the device may remain at full power.  If the device does go to low
 *	power and is capable of generating runtime wakeup events, remote wakeup
 *	(i.e., a hardware mechanism allowing the device to request a change of
 *	its power state via an interrupt) should be enabled for it.
 *
 * @runtime_resume: Put the device into the fully active state in response to a
 *	wakeup event generated by hardware or at the request of software.  If
 *	necessary, put the device into the full-power state and restore its
 *	registers, so that it is fully operational.
 *
 * @runtime_idle: Device appears to be inactive and it might be put into a
 *	low-power state if all of the necessary conditions are satisfied.
 *	Check these conditions, and return 0 if it's appropriate to let the PM
 *	core queue a suspend request for the device.
 *
 * Several device power state transitions are externally visible, affecting
 * the state of pending I/O queues and (for drivers that touch hardware)
 * interrupts, wakeups, DMA, and other hardware state.  There may also be
 * internal transitions to various low-power modes which are transparent
 * to the rest of the driver stack (such as a driver that's ON gating off
 * clocks which are not in active use).
 *
 * The externally visible transitions are handled with the help of callbacks
 * included in this structure in such a way that, typically, two levels of
 * callbacks are involved.  First, the PM core executes callbacks provided by PM
 * domains, device types, classes and bus types.  They are the subsystem-level
 * callbacks expected to execute callbacks provided by device drivers, although
 * they may choose not to do that.  If the driver callbacks are executed, they
 * have to collaborate with the subsystem-level callbacks to achieve the goals
 * appropriate for the given system transition, given transition phase and the
 * subsystem the device belongs to.
 *
 * All of the above callbacks, except for @complete(), return error codes.
 * However, the error codes returned by @resume(), @thaw(), @restore(),
 * @resume_noirq(), @thaw_noirq(), and @restore_noirq(), do not cause the PM
 * core to abort the resume transition during which they are returned.  The
 * error codes returned in those cases are only printed to the system logs for
 * debugging purposes.  Still, it is recommended that drivers only return error
 * codes from their resume methods in case of an unrecoverable failure (i.e.
 * when the device being handled refuses to resume and becomes unusable) to
 * allow the PM core to be modified in the future, so that it can avoid
 * attempting to handle devices that failed to resume and their children.
 *
 * It is allowed to unregister devices while the above callbacks are being
 * executed.  However, a callback routine MUST NOT try to unregister the device
 * it was called for, although it may unregister children of that device (for
 * example, if it detects that a child was unplugged while the system was
 * asleep).
 *
 * There also are callbacks related to runtime power management of devices.
 * Again, as a rule these callbacks are executed by the PM core for subsystems
 * (PM domains, device types, classes and bus types) and the subsystem-level
 * callbacks are expected to invoke the driver callbacks.  Moreover, the exact
 * actions to be performed by a device driver's callbacks generally depend on
 * the platform and subsystem the device belongs to.
 *
 * Refer to Documentation/power/runtime_pm.rst for more information about the
 * role of the @runtime_suspend(), @runtime_resume() and @runtime_idle()
 * callbacks in device runtime power management.
 */
struct dev_pm_ops {
	int (*prepare)(struct device *dev);
	void (*complete)(struct device *dev);
	int (*suspend)(struct device *dev);
	int (*resume)(struct device *dev);
	int (*freeze)(struct device *dev);
	int (*thaw)(struct device *dev);
	int (*poweroff)(struct device *dev);
	int (*restore)(struct device *dev);
	int (*suspend_late)(struct device *dev);
	int (*resume_early)(struct device *dev);
	int (*freeze_late)(struct device *dev);
	int (*thaw_early)(struct device *dev);
	int (*poweroff_late)(struct device *dev);
	int (*restore_early)(struct device *dev);
	int (*suspend_noirq)(struct device *dev);
	int (*resume_noirq)(struct device *dev);
	int (*freeze_noirq)(struct device *dev);
	int (*thaw_noirq)(struct device *dev);
	int (*poweroff_noirq)(struct device *dev);
	int (*restore_noirq)(struct device *dev);
	int (*runtime_suspend)(struct device *dev);
	int (*runtime_resume)(struct device *dev);
	int (*runtime_idle)(struct device *dev);
};
```

device pm calssbacks在设备模型中的体现，在linux设备模型中的很多数据结构都会包含`struct dev_pm_ops`变量
```c
// include/linux/device/driver.h
struct device_driver {
	...
	struct dev_pm_ops *pm;
	...
};

// include/linux/device/bus.h
struct bus_type {
	...
	struct dev_pm_ops *pm;
	...
};

// include/linux/device/class.h
struct class {
	...
	struct dev_pm_ops *pm;
	...
};
```

device pm callbacks的操作函数：内核在定义device pm callbacks数据结构的同时，为了方便使用该数据结构，也定义了大量的操作API，这些API分为两类
+ 通用的辅助性质的API，直接调用指定device所绑定driver的pm指针相应的callback
```c
// include/linux/pm.h
extern int pm_generic_prepare(struct device *dev);
extern int pm_generic_suspend_late(struct device *dev);
extern int pm_generic_suspend_noirq(struct device *dev);
extern int pm_generic_suspend(struct device *dev);
extern int pm_generic_resume_early(struct device *dev);
extern int pm_generic_resume_noirq(struct device *dev);
extern int pm_generic_resume(struct device *dev);
extern int pm_generic_freeze_noirq(struct device *dev);
extern int pm_generic_freeze_late(struct device *dev);
extern int pm_generic_freeze(struct device *dev);
extern int pm_generic_thaw_noirq(struct device *dev);
extern int pm_generic_thaw_early(struct device *dev);
extern int pm_generic_thaw(struct device *dev);
extern int pm_generic_restore_noirq(struct device *dev);
extern int pm_generic_restore_early(struct device *dev);
extern int pm_generic_restore(struct device *dev);
extern int pm_generic_poweroff_noirq(struct device *dev);
extern int pm_generic_poweroff_late(struct device *dev);
extern int pm_generic_poweroff(struct device *dev);
extern void pm_generic_complete(struct device *dev);

// 以pm_generic_suspend()为例，其他的接口与之类似
// driver/base/power/generic_ops.c
/**
 * pm_generic_suspend - Generic suspend callback for subsystems.
 * @dev: Device to suspend.
 */
int pm_generic_suspend(struct device *dev)
{
	const struct dev_pm_ops *pm = dev->driver ? dev->driver->pm : NULL;

	return pm && pm->suspend ? pm->suspend(dev) : 0;
}
EXPORT_SYMBOL_GPL(pm_generic_suspend);
```
+ 和整体电源管理行为相关的API
```c
// drivers/base/power/main.c
/* 
 * 设备模型在添加设备(device_add())时，会调用device_pm_add接口
 * 将该设备添加到全局链表dpm_list中，以方便后续的遍历操作
 */
LIST_HEAD(dpm_list);
static LIST_HEAD(dpm_prepared_list);
static LIST_HEAD(dpm_suspended_list);
static LIST_HEAD(dpm_late_early_list);
static LIST_HEAD(dpm_noirq_list);

// include/linux/pm.h
// 为dpm_list_mtx加锁
extern void device_pm_lock(void);
extern void dpm_resume_start(pm_message_t state);
extern void dpm_resume_end(pm_message_t state);
extern void dpm_resume_noirq(pm_message_t state);
extern void dpm_resume_early(pm_message_t state);
extern void dpm_resume(pm_message_t state);
extern void dpm_complete(pm_message_t state);

extern void device_pm_unlock(void);
extern int dpm_suspend_end(pm_message_t state);
extern int dpm_suspend_start(pm_message_t state);
extern int dpm_suspend_noirq(pm_message_t state);
extern int dpm_suspend_late(pm_message_t state);
extern int dpm_suspend(pm_message_t state);
extern int dpm_prepare(pm_message_t state);

extern void __suspend_report_result(const char *function, void *fn, int ret);

#define suspend_report_result(fn, ret)					\
	do {								\
		__suspend_report_result(__func__, fn, ret);		\
	} while (0)

extern int device_pm_wait_for_dev(struct device *sub, struct device *dev);
extern void dpm_for_each_dev(void *data, void (*fn)(struct device *, void *));
```

### 内核电源状态
在linux内核中，将电源划分为如下几个状态
| ACPI | State | linux State Drscription |
| :-: | :-: | :- |
| S0 | On | Working |
| S1 | Standby | CPU and RAM are powered but not executed |
| S2 | N/A | N/A |
| S3 | Suspend to RAM | CPU is off, RAM is powered and the running content is saved to RAM |
| S4 | Suspend to Disk | All content is saved to Disk and power down |
| S5 | Shutdown | Shutdown the system |

### Generic PM
Generic PM指的是linux系统中常规的电源管理，包括关机、待机、重启等

Generic PM的软件层次：
+ API Layer：用于向用户空间提供接口，其中关机、重启的接口形式是系统调用，Hibernate、Suspend的接口形式是sysfs
+ PM Core：位于`kernel/power`目录下，主要处理和硬件无关的核心逻辑
+ PM Driver：分为两个部分，一个是和体系结构无关的Driver，提供pm框架；另一部分是具体的体系结构相关的driver，也是开发者关注的部分

#### Generic PM软件架构


#### 用户空间接口
generic PM的用户


#### Generic PM reboot的过程

kernel支持的reboot的方式定义在`include/uapi/linux/reboot.h`中，代码如下：
```c
// include/uapi/linux/reboot.h
/*
 * Magic values required to use _reboot() system call.
 */

#define	linux_REBOOT_MAGIC1	0xfee1dead
#define	linux_REBOOT_MAGIC2	672274793
#define	linux_REBOOT_MAGIC2A	85072278
#define	linux_REBOOT_MAGIC2B	369367448
#define	linux_REBOOT_MAGIC2C	537993216


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
 * KEXEC       Restart system using a previously loaded linux kernel
 */

#define	linux_REBOOT_CMD_RESTART	0x01234567
#define	linux_REBOOT_CMD_HALT		0xCDEF0123
#define	linux_REBOOT_CMD_CAD_ON		0x89ABCDEF
#define	linux_REBOOT_CMD_CAD_OFF	0x00000000
#define	linux_REBOOT_CMD_POWER_OFF	0x4321FEDC
#define	linux_REBOOT_CMD_RESTART2	0xA1B2C3D4
#define	linux_REBOOT_CMD_SW_SUSPEND	0xD000FCE2
#define	linux_REBOOT_CMD_KEXEC		0x45584543
```

##### reboot过程的代码分析
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
	if (magic1 != linux_REBOOT_MAGIC1 ||
			(magic2 != linux_REBOOT_MAGIC2 &&
			magic2 != linux_REBOOT_MAGIC2A &&
			magic2 != linux_REBOOT_MAGIC2B &&
			magic2 != linux_REBOOT_MAGIC2C))
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
	if ((cmd == linux_REBOOT_CMD_POWER_OFF) && !kernel_can_power_off())
		cmd = linux_REBOOT_CMD_HALT;

	mutex_lock(&system_transition_mutex);
	switch (cmd) {
	case linux_REBOOT_CMD_RESTART:
		kernel_restart(NULL);
		break;

	case linux_REBOOT_CMD_CAD_ON:
		C_A_D = 1;
		break;

	case linux_REBOOT_CMD_CAD_OFF:
		C_A_D = 0;
		break;

	case linux_REBOOT_CMD_HALT:
		kernel_halt();
		do_exit(0);

	case linux_REBOOT_CMD_POWER_OFF:
		kernel_power_off();
		do_exit(0);
		break;

	case linux_REBOOT_CMD_RESTART2:
		ret = strncpy_from_user(&buffer[0], arg, sizeof(buffer) - 1);
		if (ret < 0) {
			ret = -EFAULT;
			break;
		}
		buffer[sizeof(buffer) - 1] = '\0';

		kernel_restart(buffer);
		break;

#ifdef CONFIG_KEXEC_CORE
	case linux_REBOOT_CMD_KEXEC:
		ret = kernel_kexec();
		break;
#endif

#ifdef CONFIG_HIBERNATION
	case linux_REBOOT_CMD_SW_SUSPEND:
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

向`restart_prep_handler_list`通知链发送消息
```c
/*
 *	Notifier list for kernel code which wants to be called
 *	to prepare system for restart.
 */
static BLOCKING_NOTIFIER_HEAD(restart_prep_handler_list);

static void do_kernel_restart_prepare(void)
{
	blocking_notifier_call_chain(&restart_prep_handler_list, 0, NULL);
}
```

#### suspend过程分析
linux内核提供了三种suspend: freeze、standby和STR(Suspend to RAM)，在用户空间向`/sys/power/state`文件分别写入freeze、standby、mem，即可触发它们



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
linux内核抽象出来power supply子系统为驱动提供了统一的框架，功能包括：
1. 抽象power supply设备的共性，向用户空间提供统一的API
2. 为底层power supply驱动的编写，提供简单统一的方式，同时封装并实现公共逻辑

power supply class位于`driver/power/supply`目录中，主要由3部分组成：
1. power supply core：用于抽象核心数据结构，实现公共逻辑
2. power supply sysfs：实现sysfs以及uevent功能
3. power supply leds：给予linux led class，提供power supply设备状态指示的通用实现

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
