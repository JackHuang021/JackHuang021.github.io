---
title: Linux Cpufreq 框架
tags:
  - Linux
  - Cpufreq
categories: Linux
abbrlink: fbf46cf3
date: 2022-11-24 10:23:01
---


Linux Kernel主要通过三类机制来实现SMP（Symmetric Multiprocessing，对称多核）系统CPU core的电源管理：
+ cpu hotplug: 根据应用场景来up/down CPU
+ cpuidle framework: 
+ cpufreq framework: 根据使用场景和系统负荷来调整CPU的电压和频率

<!-- more -->

> 由于调整是在系统运行的过程中，因此cpufreq framework的功能也称作为Dynamic Voltage/Frequency Scaling（动态电压/频率调整）

cpufreq framework中的几个重要概念：
1. policy（策略）：选择合适的调频范围
2. governor（调节器）：决定如何计算合适的功率
3. cpufreq driver：来实现真正的调频工作（平台相关）

常用的governor类型
1. Performance：总是将CPU置于最高性能的状态，即硬件所支持的最高频率、电压
2. Powersaving：总是将CPU置于最节能的状态，即硬件所支持的最低频率、电压
3. Ondemand：设置CPU负载的阈值T，当负载低于T时，调节至一个刚好能够满足当前负载需求的最低频/最低压；当负载高于T时，立即提升到最高性能状态
4. Conservative：跟Ondemand策略类似，设置CPU负载的阈值T，当负载低于T时，调节至一个刚好能够满足当前负载需求的最低频/最低压；但当负载高于T时，不是立即设置为最高性能状态，而是逐级升高主频/电压
5. Userspace：将控制接口通过sysfs开放给用户，由用户进行自定义策略
6. Schedutil：这是从Linux-4.7版本开始才引入的策略，其原理是根据调度器所提供的CPU利用率信息进行电压/频率调节，效果上类似于Ondemand策略，但是更加精确和自然

sysfs用户层接口，目录位于`/sys/devices/system/cpu/cpufreq/policy`
![](https://raw.githubusercontent.com/JackHuang021/images/master/20221124110212.png)

| 名称 | 说明 |
| :-- | :-- |
| cpuinfo_max_freq | 硬件所支持的最高频率 |
| cpuinfo_min_freq | 硬件所支持的最低频率 |
| affected_cpus | 该policy影响到哪些cpu（没显示offline状态的cpu）|
| related_cpus | 该policy影响到的所有cpu，包括offline状态的cpu |
| scaling_max_freq | 该policy支持调整的最高频率 |
| scaling_min_freq | 该policy支持调整的最低频率 |
| scaling_cur_freq | policy当前设置的频率 |
| scaling_available_governors | 当前系统支持的governor |
| scaling_available_frequencies | 支持的调频频率 |
| scaling_driver | 当前使用的调频驱动 |
| scaling_governor | 当前使用的governor |
| scaling_setspeed | 在userspace模式下才能使用，手动设置频率 |

#### cpufreq软件架构
cpufreq core：把一些公共的逻辑和接口代码抽象出来
+ `cpufreq`作为所有cpu设备的一个功能，注册到了`cpu_subsys`总线上
+ 对上以sysfs的形式向用户空间提供统一的接口，以notifier的形式向其他driver提供频率变化的通知
+ 对下提供CPU品频率和电压控制的驱动框架，方便底层driver的开发，同时提供governor框架，用于实现不同的频率调整机制
+ 内部封装各种逻辑，主要围绕`struct cpufreq_policy` `struct cpufreq_driver` `struct cpufreq_governor`三个数据结构进行

**`struct cpufreq_policy`用来抽象cpufreq，所谓的调频策略，即频率调整的范围，它从一定程度上代表了cpufreq的属性**

`cpufreq_policy`结构体
```c
struct cpufreq_cpuinfo {
	unsigned int		max_freq;
	unsigned int		min_freq;

	/* in 10^(-9) s = nanoseconds */
	unsigned int		transition_latency;
};

struct cpufreq_policy {
	/* CPUs sharing clock, require sw coordination */
	cpumask_var_t		cpus;	/* Online CPUs only */
	cpumask_var_t		related_cpus; /* Online + Offline CPUs */
	cpumask_var_t		real_cpus; /* Related and present */

	unsigned int		shared_type; /* ACPI: ANY or ALL affected CPUs
						should set cpufreq */
	unsigned int		cpu;    /* cpu managing this policy, must be online */

	struct clk		*clk;
	struct cpufreq_cpuinfo	cpuinfo;/* see above */

	unsigned int		min;    /* in kHz */
	unsigned int		max;    /* in kHz */
	unsigned int		cur;    /* in kHz, only needed if cpufreq
					 * governors are used */
	unsigned int		restore_freq; /* = policy->cur before transition */
	unsigned int		suspend_freq; /* freq to set during suspend */

	unsigned int		policy; /* see above */
	unsigned int		last_policy; /* policy before unplug */
	struct cpufreq_governor	*governor; /* see below */
	void			*governor_data;
	char			last_governor[CPUFREQ_NAME_LEN]; /* last governor used */

	struct work_struct	update; /* if update_policy() needs to be
					 * called, but you're in IRQ context */

	struct cpufreq_user_policy user_policy;
	struct cpufreq_frequency_table	*freq_table;
	enum cpufreq_table_sorting freq_table_sorted;

	struct list_head        policy_list;
	struct kobject		kobj;
	struct completion	kobj_unregister;

	/*
	 * The rules for this semaphore:
	 * - Any routine that wants to read from the policy structure will
	 *   do a down_read on this semaphore.
	 * - Any routine that will write to the policy structure and/or may take away
	 *   the policy altogether (eg. CPU hotplug), will hold this lock in write
	 *   mode before doing so.
	 */
	struct rw_semaphore	rwsem;

	/*
	 * Fast switch flags:
	 * - fast_switch_possible should be set by the driver if it can
	 *   guarantee that frequency can be changed on any CPU sharing the
	 *   policy and that the change will affect all of the policy CPUs then.
	 * - fast_switch_enabled is to be set by governors that support fast
	 *   frequency switching with the help of cpufreq_enable_fast_switch().
	 */
	bool			fast_switch_possible;
	bool			fast_switch_enabled;

	/*
	 * Preferred average time interval between consecutive invocations of
	 * the driver to set the frequency for this policy.  To be set by the
	 * scaling driver (0, which is the default, means no preference).
	 */
	unsigned int		transition_delay_us;

	/*
	 * Remote DVFS flag (Not added to the driver structure as we don't want
	 * to access another structure from scheduler hotpath).
	 *
	 * Should be set if CPUs can do DVFS on behalf of other CPUs from
	 * different cpufreq policies.
	 */
	bool			dvfs_possible_from_any_cpu;

	 /* Cached frequency lookup from cpufreq_driver_resolve_freq. */
	unsigned int cached_target_freq;
	int cached_resolved_idx;

	/* Synchronization for frequency transitions */
	bool			transition_ongoing; /* Tracks transition status */
	spinlock_t		transition_lock;
	wait_queue_head_t	transition_wait;
	struct task_struct	*transition_task; /* Task which is doing the transition */

	/* cpufreq-stats */
	struct cpufreq_stats	*stats;

	/* For cpufreq driver's internal use */
	void			*driver_data;
};
```

#### cpufreq_driver初始化过程
`cpufreq_driver`结构体如下：
```c
struct cpufreq_driver {
	char		name[CPUFREQ_NAME_LEN];
	u16		    flags;
	void		*driver_data;

	/* needed by all drivers */
	int		(*init)(struct cpufreq_policy *policy);
	int		(*verify)(struct cpufreq_policy_data *policy);

	/* define one out of two */
	int		(*setpolicy)(struct cpufreq_policy *policy);

	/*
	 * On failure, should always restore frequency to policy->restore_freq
	 * (i.e. old freq).
	 */
	int		(*target)(struct cpufreq_policy *policy,
				  unsigned int target_freq,
				  unsigned int relation);	/* Deprecated */
	int		(*target_index)(struct cpufreq_policy *policy,
					unsigned int index);
	unsigned int	(*fast_switch)(struct cpufreq_policy *policy,
				       unsigned int target_freq);

	/*
	 * Caches and returns the lowest driver-supported frequency greater than
	 * or equal to the target frequency, subject to any driver limitations.
	 * Does not set the frequency. Only to be implemented for drivers with
	 * target().
	 */
	unsigned int	(*resolve_freq)(struct cpufreq_policy *policy,
					unsigned int target_freq);

	/*
	 * Only for drivers with target_index() and CPUFREQ_ASYNC_NOTIFICATION
	 * unset.
	 *
	 * get_intermediate should return a stable intermediate frequency
	 * platform wants to switch to and target_intermediate() should set CPU
	 * to that frequency, before jumping to the frequency corresponding
	 * to 'index'. Core will take care of sending notifications and driver
	 * doesn't have to handle them in target_intermediate() or
	 * target_index().
	 *
	 * Drivers can return '0' from get_intermediate() in case they don't
	 * wish to switch to intermediate frequency for some target frequency.
	 * In that case core will directly call ->target_index().
	 */
	unsigned int	(*get_intermediate)(struct cpufreq_policy *policy,
					    unsigned int index);
	int		(*target_intermediate)(struct cpufreq_policy *policy,
					       unsigned int index);

	/* should be defined, if possible */
	unsigned int	(*get)(unsigned int cpu);

	/* Called to update policy limits on firmware notifications. */
	void		(*update_limits)(unsigned int cpu);

	/* optional */
	int		(*bios_limit)(int cpu, unsigned int *limit);

	int		(*online)(struct cpufreq_policy *policy);
	int		(*offline)(struct cpufreq_policy *policy);
	int		(*exit)(struct cpufreq_policy *policy);
	void		(*stop_cpu)(struct cpufreq_policy *policy);
	int		(*suspend)(struct cpufreq_policy *policy);
	int		(*resume)(struct cpufreq_policy *policy);

	/* Will be called after the driver is fully initialized */
	void		(*ready)(struct cpufreq_policy *policy);

	struct freq_attr **attr;

	/* platform specific boost support code */
	bool		boost_enabled;
	int		(*set_boost)(struct cpufreq_policy *policy, int state);
};
```

`cpufreq_register_driver()`函数为cpufreq驱动注册的入口，驱动程序通过调用该函数进行初始化，传入相关的`struct cpufreq_driver`，`cpufreq_register_driver()`会调用`subsys_interface_register()`最终执行回调函数`cpufreq_add_dev`

```c
// driver/base/cpu.c
struct bus_type cpu_subsys = {
	.name = "cpu",
	.dev_name = "cpu",
	.match = cpu_subsys_match,
	.online = cpu_subsys_online,
	.offline = cpu_subsys_offline,
};

static struct cpufreq_driver *cpufreq_driver;

static struct subsys_interface cpufreq_interface = {
	.name		= "cpufreq",
	.subsys		= &cpu_subsys,
	.add_dev	= cpufreq_add_dev,
	.remove_dev	= cpufreq_remove_dev,
};

static struct cpufreq_driver scmi_cpufreq_driver = {
	.name	= "scmi",
	.flags	= CPUFREQ_STICKY | CPUFREQ_HAVE_GOVERNOR_PER_POLICY |
		  CPUFREQ_NEED_INITIAL_FREQ_CHECK,
	.verify	= cpufreq_generic_frequency_table_verify,
	.attr	= cpufreq_generic_attr,
	.target_index	= scmi_cpufreq_set_target,
	.fast_switch	= scmi_cpufreq_fast_switch,
	.get	= scmi_cpufreq_get_rate,
	.init	= scmi_cpufreq_init,
	.exit	= scmi_cpufreq_exit,
	.ready	= scmi_cpufreq_ready,
};

// 为每个CPU定义一个cpufreq_policy结构体，对每个CPU进行调频管理
// 其中又分别进行cpufreq_driver的初始化和governor的初始化
static DEFINE_PER_CPU(struct cpufreq_policy *, cpufreq_cpu_data);

// cpufreq驱动框架初始化过程，整个过程都围绕着policy这个结构体进行，逐步进行初始化
cpufreq_register_driver(&scmi_cpufreq_driver);
	subsys_interface_register(&cpufreq_interface);
		cpufreq_add_dev(dev, sif);
			cpufreq_online(cpu);
				policy = cpufreq_policy_alloc(cpu);
				// 调用驱动init接口，初始化policy结构体
				cpufreq_driver->init(policy) -> scmi_cpufreq_init(policy)
				cpufreq_table_validate_and_sort(policy);
				add_cpu_dev_symlink();
				freq_qos_and_request();
				blocking_notifier_call_chain();
				// CPU进行频率调整，使当前运行频率在频率表中
				__cpufreq_driver_target();
				// 创建sys节点
				cpufreq_add_dev_interface(policy);
				cpufreq_stats_create_table(policy);
				list_add(&policy->polic_list, &cpufreq_policy_list);
				// 初始化governor
				cpufreq_init_policy();
```

cpufreq_driver的初始化过程，这里是调用cpufreq_driver的init接口进行初始化
```c
// scmi_cpufreq_init(policy)初始化过程
/**
 * struct scmi_handle - Handle returned to ARM SCMI clients for usage.
 *
 * @dev: pointer to the SCMI device
 * @version: pointer to the structure containing SCMI version information
 * @power_ops: pointer to set of power protocol operations
 * @perf_ops: pointer to set of performance protocol operations
 * @clk_ops: pointer to set of clock protocol operations
 * @sensor_ops: pointer to set of sensor protocol operations
 * @reset_ops: pointer to set of reset protocol operations
 * @notify_ops: pointer to set of notifications related operations
 * @perf_priv: pointer to private data structure specific to performance
 *	protocol(for internal use only)
 * @clk_priv: pointer to private data structure specific to clock
 *	protocol(for internal use only)
 * @power_priv: pointer to private data structure specific to power
 *	protocol(for internal use only)
 * @sensor_priv: pointer to private data structure specific to sensors
 *	protocol(for internal use only)
 * @reset_priv: pointer to private data structure specific to reset
 *	protocol(for internal use only)
 * @notify_priv: pointer to private data structure specific to notifications
 *	(for internal use only)
 */
struct scmi_handle {
	struct device *dev;
	struct scmi_revision_info *version;
	const struct scmi_perf_ops *perf_ops;
	const struct scmi_clk_ops *clk_ops;
	const struct scmi_power_ops *power_ops;
	const struct scmi_sensor_ops *sensor_ops;
	const struct scmi_reset_ops *reset_ops;
	const struct scmi_notify_ops *notify_ops;
	/* for protocol internal use */
	void *perf_priv;
	void *clk_priv;
	void *power_priv;
	void *sensor_priv;
	void *reset_priv;
	void *notify_priv;
	void *system_priv;
};

/**
 * struct scmi_perf_ops - represents the various operations provided
 *	by SCMI Performance Protocol
 *
 * @limits_set: sets limits on the performance level of a domain
 * @limits_get: gets limits on the performance level of a domain
 * @level_set: sets the performance level of a domain
 * @level_get: gets the performance level of a domain
 * @device_domain_id: gets the scmi domain id for a given device
 * @transition_latency_get: gets the DVFS transition latency for a given device
 * @device_opps_add: adds all the OPPs for a given device
 * @freq_set: sets the frequency for a given device using sustained frequency
 *	to sustained performance level mapping
 * @freq_get: gets the frequency for a given device using sustained frequency
 *	to sustained performance level mapping
 * @est_power_get: gets the estimated power cost for a given performance domain
 *	at a given frequency
 */
struct scmi_perf_ops {
	int (*limits_set)(const struct scmi_handle *handle, u32 domain,
			  u32 max_perf, u32 min_perf);
	int (*limits_get)(const struct scmi_handle *handle, u32 domain,
			  u32 *max_perf, u32 *min_perf);
	int (*level_set)(const struct scmi_handle *handle, u32 domain,
			 u32 level, bool poll);
	int (*level_get)(const struct scmi_handle *handle, u32 domain,
			 u32 *level, bool poll);
	int (*device_domain_id)(struct device *dev);
	int (*transition_latency_get)(const struct scmi_handle *handle,
				      struct device *dev);
	int (*device_opps_add)(const struct scmi_handle *handle,
			       struct device *dev);
	int (*freq_set)(const struct scmi_handle *handle, u32 domain,
			unsigned long rate, bool poll);
	int (*freq_get)(const struct scmi_handle *handle, u32 domain,
			unsigned long *rate, bool poll);
	int (*est_power_get)(const struct scmi_handle *handle, u32 domain,
			     unsigned long *rate, unsigned long *power);
	bool (*fast_switch_possible)(const struct scmi_handle *handle,
				     struct device *dev);
};

struct scmi_opp {
	u32 perf;
	u32 power;
	u32 trans_latency_us;
};

// scmi_handle这个结构体实现的就是scmi整个协议的处理
static const struct scmi_handle *handle;	

scmi_cpufreq_init(policy)
	cpu_dev = get_cpu_device(policy->cpu);
	handle->perf_ops->device_opps_add(handle, cpu_dev);
```

cpufreq governor的初始化过程，在cpufreq_init_policy(policy)中进行，这里以ondemand为例进行分析
```c
/* Common Governor data across policies */
struct dbs_governor {
	struct cpufreq_governor gov;
	struct kobj_type kobj_type;

	/*
	 * Common data for platforms that don't set
	 * CPUFREQ_HAVE_GOVERNOR_PER_POLICY
	 */
	struct dbs_data *gdbs_data;

	unsigned int (*gov_dbs_update)(struct cpufreq_policy *policy);
	struct policy_dbs_info *(*alloc)(void);
	void (*free)(struct policy_dbs_info *policy_dbs);
	int (*init)(struct dbs_data *dbs_data);
	void (*exit)(struct dbs_data *dbs_data);
	void (*start)(struct cpufreq_policy *policy);
};

#define CPUFREQ_DBS_GOVERNOR_INITIALIZER(_name_)			\
	{								\
		.name = _name_,						\
		.flags = CPUFREQ_GOV_DYNAMIC_SWITCHING,			\
		.owner = THIS_MODULE,					\
		.init = cpufreq_dbs_governor_init,			\
		.exit = cpufreq_dbs_governor_exit,			\
		.start = cpufreq_dbs_governor_start,			\
		.stop = cpufreq_dbs_governor_stop,			\
		.limits = cpufreq_dbs_governor_limits,			\
	}

static struct dbs_governor od_dbs_gov = {
	.gov = CPUFREQ_DBS_GOVERNOR_INITIALIZER("ondemand"),
	.kobj_type = { .default_attrs = od_attributes },
	.gov_dbs_update = od_dbs_update,
	.alloc = od_alloc,
	.free = od_free,
	.init = od_init,
	.exit = od_exit,
	.start = od_start,
};

#define CPU_FREQ_GOV_ONDEMAND	(od_dbs_gov.gov)

// 从这里开始初始化governor
// 该函数会在所有governor模块驱动的入口函数调用，
// 只要编译该模块，就会注册到cpufreq framework中
cpufreq_governor_init(CPU_FREQ_GOV_ONDEMAND);

cpufreq_init_policy(policy);
	gov = get_governor(default_governor);
	// 设置新的governor
	cpufreq_set_policy(policy, gov, pol);
		cpufreq_init_governor(policy);
			policy->governor->init();  -> cpufreq_dbs_governor_init(policy);
				gov->init(dbs_data); -> odinit(dbs_data);
		cpufreq_start_governor(policy);
			policy->governor->start(); -> cpufreq_dbs_governor_start(policy);
				// 设置governor回调函数
				gov_set_update_util(policy_dbs, sampling_rate);
					cpufreq_add_update_util_hook(cpu, &cdbs->updata_util,
									dbs_update_util_handler);
```
启动governor中比较重要的是设置调频回调函数,该函数是真正调频时计算合适频率的函数

`cpufreq_driver`必须要实现的几个接口：
```c
int (*verify)(struct cpufreq_policy *policy);
int (*init)(struct cpufreq_policy *policy);
int (*setpolicy)(struct cpufreq_policy *policy)
int (*target_index)(struct cpufreq_policy *policy, unsigned int index); 
```

```c
static struct subsys_interface cpufreq_interface = {
	.name		= "cpufreq",
	.subsys		= &cpu_subsys,
	.add_dev	= cpufreq_add_dev,
	.remove_dev	= cpufreq_remove_dev,
};

// cpufreq driver注册过程
cpufreq_register_driver(&scmi_cpufreq_driver);
    subsys_interface_register(&cpufreq_interface);
        cpufreq_add_dev(dev, cpu_subsys);
            cpufreq_online(cpu);
```


由于bootloader设置的CPU频率值可能不在当前CPU频率表中，CPU在该频率下可能运行不稳定，所以在cpufreq_online(cpu)过程中会进行一次调频


schedutil调度器代码分析

sugov作为一种内核调频策略模块，它主要是根据当前CPU的利用率进行调频。因此，sugov会注册一个callback函数（sugov_update_shared/sugov_update_single）到调度器负载跟踪模块，当CPU util发生变化的时候就会调用该callback函数，检查一下当前CPU频率是否和当前的CPU util匹配，如果不匹配，那么就进行提频或者降频。

`sugov_tunables`结构体，用来描述sugov的可调参数
```c
// sugov_tunables结构体
struct sugov_tunables {
	struct gov_attr_set attr_set;
	// 目前sugov只有这一个可调参数，该参数用来限制连续调频的间隔时间
	unsigned int rate_limit_us;
};
```

`sugov_policy`结构体，sugov为每个cluster构建了该数据结构，记录per-cluster的调频数据信息
```c
// sugov_policy结构体
struct sugov_policy {
	// 指向cpufreq framework层的policy对象
	struct cpufreq_policy	*policy;
	// sugov的可调参数
	struct sugov_tunables	*tunables;
	struct list_head	tunables_hook;

	// 保护sugov对象的自旋锁，一旦要修改
	raw_spinlock_t		update_lock;	/* For shared policies */
	u64			last_freq_update_time;
	s64			freq_update_delay_ns;
	unsigned int		next_freq;
	unsigned int		cached_raw_freq;

	/* The next fields are only needed if fast switch cannot be used: */
	struct			irq_work irq_work;
	struct			kthread_work work;
	struct			mutex work_lock;
	struct			kthread_worker worker;
	struct task_struct	*thread;
	bool			work_in_progress;

	bool			limits_changed;
	bool			need_freq_update;
};
```

`sugov_cpu`结构体，sugov为每个cpu构建了该数据结构，记录per-cpu的调频数据信息
```c
struct sugov_cpu {
	// 保存了cpu util变化后的callback函数
	struct update_util_data	update_util;
	// 该sugov_cpu对应的sugov_policy对象
	struct sugov_policy	*sg_policy;
	// 对应的CPU id
	unsigned int		cpu;

	bool			iowait_boost_pending;
	unsigned int		iowait_boost;
	// 上一次cpu负载变化驱动调频的时间点，util更新的时间点
	u64			last_update;

	unsigned long		bw_dl;
	// 该CPU的最大算力，即最大utilities，归一化到1024
	unsigned long		max;

	/* The field below is for single-CPU policies only: */
#ifdef CONFIG_NO_HZ_COMMON
	unsigned long		saved_idle_calls;
#endif
};
```


`schedutil_cpu_util()`函数分析，用来计算cpu util的
```c
// 调用过程
sugov_update_single();
	util = sugov_get_util(sg_cpuu);
		schedutil_cpu_util(sg_cpu->cpu, util, max, FREQUENCY_UTIL, NULL);


unsigned long schedutil_cpu_util(int cpu, unsigned long util_cfs,
					unsigned long max, enum schedutil_type type,
					struct task_struct *p)
{
	unsigned long dl_util, util, irq;
	struct rq *rq = cpu_rq(cpu);

	if (!uclamp_is_used() &&
	    type == FREQUENCY_UTIL && rt_rq_is_runnable(&rq->rt)) {
		return max;
	}

	/*
	 * Early check to see if IRQ/steal time saturates the CPU, can be
	 * because of inaccuracies in how we track these -- see
	 * update_irq_load_avg().
	 */
	irq = cpu_util_irq(rq);
	if (unlikely(irq >= max))
		return max;

	/*
	 * Because the time spend on RT/DL tasks is visible as 'lost' time to
	 * CFS tasks and we use the same metric to track the effective
	 * utilization (PELT windows are synchronized) we can directly add them
	 * to obtain the CPU's actual utilization.
	 *
	 * CFS and RT utilization can be boosted or capped, depending on
	 * utilization clamp constraints requested by currently RUNNABLE
	 * tasks.
	 * When there are no CFS RUNNABLE tasks, clamps are released and
	 * frequency will be gracefully reduced with the utilization decay.
	 */
	// 累加了cfs和rt任务的utility，根据当前的
	util = util_cfs + cpu_util_rt(rq);
	if (type == FREQUENCY_UTIL)
		util = uclamp_rq_util_with(rq, util, p);

	dl_util = cpu_util_dl(rq);

	/*
	 * For frequency selection we do not make cpu_util_dl() a permanent part
	 * of this sum because we want to use cpu_bw_dl() later on, but we need
	 * to check if the CFS+RT+DL sum is saturated (ie. no idle time) such
	 * that we select f_max when there is no idle time.
	 *
	 * NOTE: numerical errors or stop class might cause us to not quite hit
	 * saturation when we should -- something for later.
	 */
	if (util + dl_util >= max)
		return max;

	/*
	 * OTOH, for energy computation we need the estimated running time, so
	 * include util_dl and ignore dl_bw.
	 */
	if (type == ENERGY_UTIL)
		util += dl_util;

	/*
	 * There is still idle time; further improve the number by using the
	 * irq metric. Because IRQ/steal time is hidden from the task clock we
	 * need to scale the task numbers:
	 *
	 *              max - irq
	 *   U' = irq + --------- * U
	 *                 max
	 */
	util = scale_irq_capacity(util, irq, max);
	util += irq;

	/*
	 * Bandwidth required by DEADLINE must always be granted while, for
	 * FAIR and RT, we use blocked utilization of IDLE CPUs as a mechanism
	 * to gracefully reduce the frequency when no tasks show up for longer
	 * periods of time.
	 *
	 * Ideally we would like to set bw_dl as min/guaranteed freq and util +
	 * bw_dl as requested freq. However, cpufreq is not yet ready for such
	 * an interface. So, we only do the latter for now.
	 */
	if (type == FREQUENCY_UTIL)
		util += cpu_bw_dl(rq);

	return min(max, util);
}
```

