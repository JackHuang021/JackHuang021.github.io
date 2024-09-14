---
title: Linux reboot流程
abbrlink: 3fca1565
date: 2023-08-11 17:55:25
tags:
---

## 1. reboot系统调用
kernel根据不同的表现方式，将reboot分为如下的几种方式：
<!-- more -->
```c
// include/uapi/linux/reboot.h
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _UAPI_LINUX_REBOOT_H
#define _UAPI_LINUX_REBOOT_H

/*
 * Magic values required to use _reboot() system call.
 */
// 重启的魔数，会在系统调用时传入
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

#endif /* _UAPI_LINUX_REBOOT_H */
```

1. RESTART: 正常的重启
2. HALT: 停止操作系统
3. CAD_ON/CAD_OFF: 允许/禁止通过Ctrl+Alt+Del组合按键触发重启动作
4. POWER_OFF: 正常的关机
5. RESTART2: 重启的另一种方式，可以在重启时携带一个字符串类型的cmd，该cmd会在重启前，发送给任意一个关心重启事件的进程，同时会传递给最终执行重启动作的machine相关的代码


在linux操作系统中，可以通过reboot、halt、poweroff等命令，发起reboot。reboot系统调用，magic1和magic2为两个魔数，防止误操作
```c
// kernel/reboot.c
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
	// 判断当前用户是否为root用户
	if (!ns_capable(pid_ns->user_ns, CAP_SYS_BOOT))
		return -EPERM;

	/* For safety, we require "magic" arguments. */

	// 判断magic数，防止误操作
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
	// 如果是poweroff指令，且没有注册的pm_power_off处理函数，则把该命令转为HALT
	if ((cmd == LINUX_REBOOT_CMD_POWER_OFF) && !kernel_can_power_off())
		cmd = LINUX_REBOOT_CMD_HALT;

	mutex_lock(&system_transition_mutex);
	// 根据不同的命令，执行具体的处理
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
}
```

kernel_restart()源码解析
```c
// 将task转移到一个CPU
void migrate_to_reboot_cpu(void)
{
	/* The boot cpu is always logical cpu 0 */
	int cpu = reboot_cpu;

	cpu_hotplug_disable();

	/* Make certain the cpu I'm about to reboot on is online */
	if (!cpu_online(cpu))
		cpu = cpumask_first(cpu_online_mask);

	/* Prevent races with other tasks migrating this task */
	current->flags |= PF_NO_SETAFFINITY;

	/* Make certain I only run on the appropriate processor */
	set_cpus_allowed_ptr(current, cpumask_of(cpu));
}


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
	// 进行restart前的准备工作，发送reboot事件，更新system_state变量状态
	kernel_restart_prepare(cmd);
	// 将当前的进程转移到一个CPU上
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
```

详细的流程如下：

![](https://raw.githubusercontent.com/JackHuang021/images/master/Linux +reboot.png)


## 2. PSCI电源状态协调接口(Power State Coordination Interface)
PSCI(Power State Coordination Interface)，电源状态协调接口，是ARM定义的电源管理接口规范，可以在官网上下载到相关资料[下载地址](https://developer.arm.com/documentation/den0022/latest/)，内核驱动位于`drivers/firmware/psci`

先看E2000的设备树关于psci的描述，设备树里面的信息如下里标记的版本是psci-1.0，method是使用smc。
```c
psci {
	compatible   = "arm,psci-1.0";
	method       = "smc";
	cpu_suspend  = <0xc4000001>;
	cpu_off      = <0x84000002>;
	cpu_on       = <0xc4000003>;
	sys_poweroff = <0x84000008>;
	sys_reset    = <0x84000009>;
};
```

psci的初始化流程
```bash
psci_dt_init() -> psci_1_0_init() -> psci_0_2_init() -> psci_probe()
```

E2000启动时，psci的相关打印
```bash
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.2
```

`psci_0_2_set_functions()`函数如下
```c
// drivers/firmware/psci/psci.c
static struct notifier_block psci_sys_reset_nb = {
	.notifier_call = psci_sys_reset,
	.priority = 129,
};

static void __init psci_0_2_set_functions(void)
{
	pr_info("Using standard PSCI v0.2 function IDs\n");

	psci_ops = (struct psci_operations){
		.get_version = psci_0_2_get_version,
		.cpu_suspend = psci_0_2_cpu_suspend,
		.cpu_off = psci_0_2_cpu_off,
		.cpu_on = psci_0_2_cpu_on,
		.migrate = psci_0_2_migrate,
		.affinity_info = psci_affinity_info,
		.migrate_info_type = psci_migrate_info_type,
	};
	// 注册到restart_handler_list通知链，重启的时候会调用psci_sys_reset
	register_restart_handler(&psci_sys_reset_nb);

	pm_power_off = psci_sys_poweroff;
}
```

`psci_sys_reset()`函数
```c
static int psci_sys_reset(struct notifier_block *nb, unsigned long action,
			  void *data)
{
	if ((reboot_mode == REBOOT_WARM || reboot_mode == REBOOT_SOFT) &&
	    psci_system_reset2_supported) {
		/*
		 * reset_type[31] = 0 (architectural)
		 * reset_type[30:0] = 0 (SYSTEM_WARM_RESET)
		 * cookie = 0 (ignored by the implementation)
		 */
		invoke_psci_fn(PSCI_FN_NATIVE(1_1, SYSTEM_RESET2), 0, 0, 0);
	} else {
		invoke_psci_fn(PSCI_0_2_FN_SYSTEM_RESET, 0, 0, 0);
	}

	return NOTIFY_DONE;
}
```

`PSCI_0_2_FN_SYSTEM_RESET`宏定义，这个宏的定义来源于PSCI文档
```c
/* PSCI v0.2 interface */
#define PSCI_0_2_FN_BASE			0x84000000
#define PSCI_0_2_FN(n)				(PSCI_0_2_FN_BASE + (n))
#define PSCI_0_2_64BIT				0x40000000
#define PSCI_0_2_FN64_BASE			\
					(PSCI_0_2_FN_BASE + PSCI_0_2_64BIT)
#define PSCI_0_2_FN64(n)			(PSCI_0_2_FN64_BASE + (n))

#define PSCI_0_2_FN_PSCI_VERSION		PSCI_0_2_FN(0)
#define PSCI_0_2_FN_CPU_SUSPEND			PSCI_0_2_FN(1)
#define PSCI_0_2_FN_CPU_OFF			PSCI_0_2_FN(2)
#define PSCI_0_2_FN_CPU_ON			PSCI_0_2_FN(3)
#define PSCI_0_2_FN_AFFINITY_INFO		PSCI_0_2_FN(4)
#define PSCI_0_2_FN_MIGRATE			PSCI_0_2_FN(5)
#define PSCI_0_2_FN_MIGRATE_INFO_TYPE		PSCI_0_2_FN(6)
#define PSCI_0_2_FN_MIGRATE_INFO_UP_CPU		PSCI_0_2_FN(7)
#define PSCI_0_2_FN_SYSTEM_OFF			PSCI_0_2_FN(8)
#define PSCI_0_2_FN_SYSTEM_RESET		PSCI_0_2_FN(9)
```

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240912135533.png)


继续看`invoke_psci_fn`的定义，可以看到被赋值为`__invoke_psci_fn_smc`，`__invoke_psci_fn_smc()`中再调用`arm_smccc_smc()`
```c
// drivers/firmware/psci/psci.c
static __always_inline unsigned long
__invoke_psci_fn_smc(unsigned long function_id,
		     unsigned long arg0, unsigned long arg1,
		     unsigned long arg2)
{
	struct arm_smccc_res res;

	arm_smccc_smc(function_id, arg0, arg1, arg2, 0, 0, 0, 0, &res);
	return res.a0;
}

static void set_conduit(enum arm_smccc_conduit conduit)
{
	switch (conduit) {
	case SMCCC_CONDUIT_HVC:
		invoke_psci_fn = __invoke_psci_fn_hvc;
		break;
	case SMCCC_CONDUIT_SMC:
		invoke_psci_fn = __invoke_psci_fn_smc;
		break;
	default:
		WARN(1, "Unexpected PSCI conduit %d\n", conduit);
	}

	psci_conduit = conduit;
}

static int get_set_conduit_method(const struct device_node *np)
{
	const char *method;

	pr_info("probing for conduit method from DT.\n");

	if (of_property_read_string(np, "method", &method)) {
		pr_warn("missing \"method\" property\n");
		return -ENXIO;
	}

	if (!strcmp("hvc", method)) {
		set_conduit(SMCCC_CONDUIT_HVC);
	} else if (!strcmp("smc", method)) {
		set_conduit(SMCCC_CONDUIT_SMC);
	} else {
		pr_warn("invalid \"method\" property: %s\n", method);
		return -EINVAL;
	}
	return 0;
}
```

`invoke_psci_fn(PSCI_0_2_FN_SYSTEM_RESET, 0, 0, 0);`即将`PSCI_0_2_FN_SYSTEM_RESET(0x84000009)`传入到`__invoke_psci_fn_smc()`的`function_id`后，调用arm_smccc_smc，`arm_smccc_smc`是一个宏，调用的是汇编编写的`__arm_smccc_smc`函数，函数的声明如下：
```c
// arch/arm64/kernel/smccc-call.S
/*
 * void arm_smccc_smc(unsigned long a0, unsigned long a1, unsigned long a2,
 *		  unsigned long a3, unsigned long a4, unsigned long a5,
 *		  unsigned long a6, unsigned long a7, struct arm_smccc_res *res,
 *		  struct arm_smccc_quirk *quirk)
 */
SYM_FUNC_START(__arm_smccc_smc)
	SMCCC	smc
SYM_FUNC_END(__arm_smccc_smc)
EXPORT_SYMBOL(__arm_smccc_smc)
```

其中SMCCC宏的定义如下：
```s
	.macro SMCCC instr
	stp     x29, x30, [sp, #-16]!
	mov	x29, sp
alternative_if ARM64_SVE
	bl	__arm_smccc_sve_check
alternative_else_nop_endif
	\instr	#0
	ldr	x4, [sp, #16]
	stp	x0, x1, [x4, #ARM_SMCCC_RES_X0_OFFS]
	stp	x2, x3, [x4, #ARM_SMCCC_RES_X2_OFFS]
	ldr	x4, [sp, #24]
	cbz	x4, 1f /* no quirk structure */
	ldr	x9, [x4, #ARM_SMCCC_QUIRK_ID_OFFS]
	cmp	x9, #ARM_SMCCC_QUIRK_QCOM_A6
	b.ne	1f
	str	x6, [x4, ARM_SMCCC_QUIRK_STATE_OFFS]
1:	ldp     x29, x30, [sp], #16
	ret
	.endm
```

smc指令是arm-v8手册中定义的一个指令，这个安全监视器触发一个异常，然后进入到EL3，在支持Trustzone的ARRMv8中，当在non-secure world或者secure world中触发了smc或者hvc指令时都属于产生sync类型的异常。根据产生该异常是否会导致EL的迁移从异常向量表中确定最终用于处理该异常的handle。

从reboot的日志可以看到PSCI的打印日志，这些log都是从PBF(Processor Base Firmware)中打印出来的，smc指令触发异常后跳转到了PBF进行处理，最后由uboot发送命令到cpld/fpga进行电源的下电和重新上电。
```bash
N: Standard Service Call: 0x84000009 
I: PSCI Power Domain Map:
I:   Domain Node : Level 2, parent_node -1, State ON (0x0)
I:   Domain Node : Level 1, parent_node 0, State ON (0x0)
I:   Domain Node : Level 1, parent_node 0, State ON (0x0)
I:   Domain Node : Level 1, parent_node 0, State ON (0x0)
I:   CPU Node : MPID 0x0, parent_node 1, State ON (0x0)
I:   CPU Node : MPID 0x100, parent_node 2, State ON (0x0)
I:   CPU Node : MPID 0x200, parent_node 3, State ON (0x0)
I:   CPU Node : MPID 0x201, parent_node 3, State ON (0x0)
N: System Reset
N: scmi iacc init
N: PFDI: jump to 3818a998
N: PFDI: jump into
u-boot : get pfdi : 1 , gd_base = 0x30c3f000
u-boot : send cmd to cpld : 4 
```

## 3. 看门狗复位
在E2000 demo板上进行看门狗复位，看门狗复位与通过CPU复位管脚进行复位的操作有点类似，不会走linux的reboot流程，E2000 demo板没有通过fpga控制电源，所以只能通过看门狗复位的方式来实现类似重启的功能。
```bash
root@Ubuntu:~# echo 7 > /proc/sys/kernel/printk
root@Ubuntu:~# cat /dev/watchdog0
[   62.938528] watchdog: watchdog0: watchdog did not stop!

root@Ubuntu:~# P: power on...
P: init stack done
P: check reset
N: wait for scp ready
P: chip_id : 
P: 37302e31
N: 5: c0000000; 6: c0000000; 7: c0000003; 4: 80000002
P: Booting Trusted Firmware
P: BL1: v2.3(release):E2000-v1.08
```


## 4. 参考链接
1. [arm64-reboot流程](https://blog.csdn.net/weixin_38878510/article/details/111686540)
2. [Linux电源管理(3)_Generic PM之Reboot过程](http://www.wowotech.net/pm_subsystem/reboot.html)