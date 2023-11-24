---
title: Linux Cpufreq框架
tags:
  - Linux
  - Cpufreq
categories: Linux
abbrlink: fbf46cf3
date: 2022-11-24 10:23:01
---

### cpufreq动态调频
#### cpufreq概述
Linux Kernel主要通过三类机制来实现SMP（Symmetric Multiprocessing，对称多核）系统CPU core的电源管理：
+ cpu hotplug: 根据应用场景来up/down CPU
+ cpuidle framework: 当cpu上没有可执行任务时，就会进入空闲状态
+ cpufreq framework: 根据使用场景和系统负荷来调整CPU的电压和频率

<!-- more -->
cpufreq framework的核心功能，是通过调整CPU core的电压或频率，兼顾系统的性能和功耗。在不需要高性能时，降低电压或频率，以降低功耗；在需要高性能时，提高电压或频率，以提高性能。

cpufreq framework中的几个重要概念：
1. policy（策略）：同一个power domain CPU动态调频策略，包含了当前使用的governor和cpufreq driver
2. governor（调节器）：决定如何计算合适的频率或电压
3. cpufreq driver（调频驱动）：实现真正的调频执行工作（与平台相关）

除此之外，cpufreq还包含cpufreq stats, cpufreq qos, cpufreq notifier等辅助模块，其主要功能如下：
1. cpufreq stats：用于搜集cpufreq的一些统计数据，如CPU在每个频点下的运行时间，总的频率切换次数等
2. cpufreq qos：用于cpufreq频率限制值发生改变时，向cpufreq模块发送一个通知，将频率限制值调整到新的值
3. cpufreq notifer：对CPU频率切换或policy对应governor发生改变感兴趣的模块，可以向cpufreq注册一个通知，当以上事件发生时，cpufreq将会向其发送相关通知

常用的governor类型
1. Performance：总是将CPU置于最高性能的状态，即硬件所支持的最高频率、电压
2. Powersaving：总是将CPU置于最节能的状态，即硬件所支持的最低频率、电压
3. Ondemand：设置CPU负载的阈值T，当负载低于T时，调节至一个刚好能够满足当前负载需求的最低频/最低压；当负载高于T时，立即提升到最高性能状态
4. Conservative：跟Ondemand策略类似，设置CPU负载的阈值T，当负载低于T时，调节至一个刚好能够满足当前负载需求的最低频/最低压；但当负载高于T时，不是立即设置为最高性能状态，而是逐级升高主频/电压
5. Userspace：将控制接口通过sysfs开放给用户，由用户进行自定义策略
6. Schedutil：这是从Linux-4.7版本开始才引入的策略，其原理是根据调度器所提供的CPU利用率信息进行电压/频率调节，EAS使用schedutil进行调频

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
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230103092146.png)
cpufreq core（可以理解为对policy的操作）：把一些公共的逻辑和接口代码抽象出来

+ `cpufreq`作为所有cpu设备的一个功能，注册到了`cpu_subsys`总线上
+ 对上以sysfs的形式向用户空间提供统一的接口，以notifier的形式向其他driver提供频率变化的通知
+ 对下提供CPU频率和电压控制的驱动框架，方便底层driver的开发，同时提供governor框架，用于实现不同的频率调整机制
+ 内部封装各种逻辑，主要围绕`struct cpufreq_policy` `struct cpufreq_driver` `struct cpufreq_governor`三个数据结构进行

kernel使用`struct cpufreq_policy`用来抽象cpufreq，它代表了一个CPU簇的cpufreq的属性

`cpufreq_policy`结构体
```c
struct cpufreq_cpuinfo {
	unsigned int		max_freq;			// cpu最大频率
	unsigned int		min_freq;			// cpu最小频率

	/* in 10^(-9) s = nanoseconds */
	unsigned int		transition_latency;	// cpu频率转换时间 单位：ns
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

	/* cpufreq-stats */
	struct cpufreq_stats	*stats;

	/* For cpufreq driver's internal use */
	void			*driver_data;

	...
};
```

`driver/cpufreq/cpufreq.c`中定义了一个全局的percpu变量
```c
static DEFINE_PER_CPU(struct cpufreq_policy *, cpufreq_cpu_data);
```
这里对应E2000 sysfs中3个policy文件夹，两个小核使用1个policy，另外两个大核分别对应1个policy  
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230103094857.png)

#### cpufreq初始化
##### 内核配置

在kconfig中(CPU Power Management -> CPU Frequency scaling)可以对cpufreq进行配置，可以配置支持的governor及系统默认的governor，以及cpufreq调频driver，例如Phytium E2000 5.10内核的配置如下，默认使用schedutil governor，根据调度器所提供的CPU利用率信息进行电压/频率调节，EAS能源感知依赖该governor工作：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230105110139.png)

##### OPP表初始化
OPP表的定义：域中每个设备支持的电压和频率的离散元组的集合称为Operating Performance Points（OPP）,内核设备树opp文档`Documentation/devicetree/bindings/opp/opp.txt`  

假设一个CPU设备支持如下的电压和频率关系：  
{300MHz at minimum voltage of 1V}  
{800MHz at minimum voltage of 1.2V}  
{1GHz at minimum voltage of 1.3V}  
用OPP表示就可以用{Hz,  uV}方式表示如下:  
{300000000, 1000000}  
{800000000, 1200000}  
{1000000000, 1300000}

Linux内核使用opp layer库来管理opp table，具体的结构如下：


Linux内核使用`struct dev_pm_opp`结构表示设备的一OPP
```c
// drivers/opp/opp.h
struct dev_pm_opp {
	struct list_head node;
	struct kref kref;

	bool available;
	bool dynamic;
	bool turbo;
	bool suspend;
	unsigned int pstate;
	unsigned long rate;
	unsigned int level;

	struct dev_pm_opp_supply *supplies;
	struct dev_pm_opp_icc_bw *bandwidth;

	unsigned long clock_latency_ns;

	struct dev_pm_opp **required_opps;
	struct opp_table *opp_table;

	struct device_node *np;

#ifdef CONFIG_DEBUG_FS
	struct dentry *dentry;
#endif
};
```

Linux内核opp layer库的结构如下
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230307162438.png)



这里初始化的就是各个性能域（即不同cluster）的OPP表，在E2000平台中是通过SCMI的Performace domain management protocol协议获取PERFORMANCE_DESCRIBE_LEVELS这个参数表，具体的协议实现源码在`drivers/firmware/arm_scmi/perf.c`里面，`perf.c`实现了SCMI的Performance domain managment protocol，scmi cpufreq_drvier也是通过`perf_ops`函数集进行调频
```c
// include/linux/scmi_protocol.h
// 抽象描述scmi协议的结构体，相应的ops操作集对应scmi的一个协议
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
	// perf_priv存放括performance domain数据，包含opp表等
	void *perf_priv;
	void *clk_priv;
	void *power_priv;
	void *sensor_priv;
	void *reset_priv;
	void *notify_priv;
	void *system_priv;
};

// include/linux/scmi_protocol.h
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

// scmi performance domain management protocol(性能域管理相关协议 0x13), messageid: 0x03
// scmi opp结构体定义
struct scmi_opp {
	u32 perf;		// 性能级别，单位KHz
	u32 power;		// 当前性能级别的功耗
	u32 trans_latency_us;	// 切换延时
};

// drivers/firmware/arm_scmi/perf.c
// scmi performance domain management protocol(性能域管理相关协议 0x13)对应操作函数集
// scmi cpufreq_driver 主要利用这个函数集进行调频相关操作
// 对应Performace domain management protocol各个message_id
static const struct scmi_perf_ops perf_ops = {
	.limits_set = scmi_perf_limits_set,
	.limits_get = scmi_perf_limits_get,
	.level_set = scmi_perf_level_set,
	.level_get = scmi_perf_level_get,
	.device_domain_id = scmi_dev_domain_id,
	.transition_latency_get = scmi_dvfs_transition_latency_get,
	.device_opps_add = scmi_dvfs_device_opps_add,
	.freq_set = scmi_dvfs_freq_set,
	.freq_get = scmi_dvfs_freq_get,
	.est_power_get = scmi_dvfs_est_power_get,
	.fast_switch_possible = scmi_fast_switch_possible,
};

// 在这个宏进行SCMI performance domain management protocol协议的初始化
DEFINE_SCMI_PROTOCOL_REGISTER_UNREGISTER(SCMI_PROTOCOL_PERF, perf)

// drivers/firmware/arm_scmi/common.h
#define DEFINE_SCMI_PROTOCOL_REGISTER_UNREGISTER(id, name) \
int __init scmi_##name##_register(void) \
{ \
	return scmi_protocol_register((id), &scmi_##name##_protocol_init); \
} \
\
void __exit scmi_##name##_unregister(void) \
{ \
	scmi_protocol_unregister((id)); \
}
// 展开该宏
int __init scmi_perf_register(void)
{
	return scmi_protocol_register(SCMI_PROTOCOL_PER, &scmi_perf_protocol_init);
}

// 初始化过程中调用了scmi_perf_protocol_init();
static int scmi_perf_protocol_init(struct scmi_handle *handle)
{
	int domain;
	u32 version;
	struct scmi_perf_info *pinfo;

	// 获取当前perf domain management协议版本
	scmi_version_get(handle, SCMI_PROTOCOL_PERF, &version);

	dev_dbg(handle->dev, "Performance Version %d.%d\n",
		PROTOCOL_REV_MAJOR(version), PROTOCOL_REV_MINOR(version));

	pinfo = devm_kzalloc(handle->dev, sizeof(*pinfo), GFP_KERNEL);
	if (!pinfo)
		return -ENOMEM;
	// 通过PROTOCOL_ATTRIBUTES(0x01)这个命令获取domain id个数
	// 还有share memory的地址和长度
	scmi_perf_attributes_get(handle, pinfo);

	// 分配perf_dom_info结构体内存，这里面的opp成员会存放OPP表
	// 对于E2000Q这里会分配3个大小的dom_info
	pinfo->dom_info = devm_kcalloc(handle->dev, pinfo->num_domains,
				       sizeof(*pinfo->dom_info), GFP_KERNEL);
	if (!pinfo->dom_info) 
		return -ENOMEM;

	// 遍历每个performance_domain，获取performance domain的属性和performance level参数
	for (domain = 0; domain < pinfo->num_domains; domain++) {
		struct perf_dom_info *dom = pinfo->dom_info + domain;

		// 获取performance domain属性
		scmi_perf_domain_attributes_get(handle, domain, dom);
		// 获取performance level参数即opp表
		// 填充perf_dom_info内的opp结构体
		scmi_perf_describe_levels_get(handle, domain, dom);

		if (dom->perf_fastchannels)
			scmi_perf_domain_init_fc(handle, domain, &dom->fc_info);
	}

	scmi_register_protocol_events(handle,
				      SCMI_PROTOCOL_PERF, SCMI_PROTO_QUEUE_SZ,
				      &perf_event_ops, perf_events,
				      ARRAY_SIZE(perf_events),
				      pinfo->num_domains);

	pinfo->version = version;
	handle->perf_ops = &perf_ops;
	// perf_priv保存performance domain信息，后续会用到perf_priv
	handle->perf_priv = pinfo;

	return 0;
}
```
在初始化阶段，`scmi_perf_protocol_init`只会将固件里面的perf domains信息保存到handle->perf_priv里面，此时还并没有将opp表注册到cpu设备上

接下来在scmi调频驱动初始化的过程中，会调用scmi的device_opps_add()接口初始化，即调用`scmi_dvfs_device_opps_add()`，在这个里面才会生成cpu的opp_table
```c
static int scmi_dvfs_device_opps_add(const struct scmi_handle *handle,
				     struct device *dev)
{
	int idx, ret, domain;
	unsigned long freq;
	struct scmi_opp *opp;
	struct perf_dom_info *dom;
	// 获取pref_priv地址，在这里面取opp数据
	struct scmi_perf_info *pi = handle->perf_priv;
	// 这个domain是从设备树里面取到的，设备树cpu节点的clock属性会带一个domain编号
	domain = scmi_dev_domain_id(dev);
	if (domain < 0)
		return domain;
	// 取当前CPU的perf domain信息
	dom = pi->dom_info + domain;
	// struct perf_dom_info结构体里面一共16个opp表
	// 这里没对opp进行限制，可能有数组越界的问题
	for (opp = dom->opp, idx = 0; idx < dom->opp_count; idx++, opp++) {
		freq = opp->perf * dom->mult_factor;
		// 这里面第一次会为opp_table分配内存，这里只将频率加进opp table里了
		ret = dev_pm_opp_add(dev, freq, 0);
		if (ret) {
			dev_warn(dev, "failed to add opp %luHz\n", freq);

			while (idx-- > 0) {
				freq = (--opp)->perf * dom->mult_factor;
				dev_pm_opp_remove(dev, freq);
			}
			return ret;
		}
	}
	return 0;
}
```

详细看一下`dev_pm_opp_add()`的过程
```c
// drviers/opp/opp.h
// opp_table结构体的定义
struct opp_table {
	struct list_head node;

	struct blocking_notifier_head head;
	struct list_head dev_list;			// 共享该opp表的设备链表
	struct list_head opp_list;
	struct kref kref;
	struct mutex lock;

	struct device_node *np;
	unsigned long clock_latency_ns_max;

	/* For backward compatibility with v1 bindings */
	unsigned int voltage_tolerance_v1;

	unsigned int parsed_static_opps;
	enum opp_table_access shared_opp;
	struct dev_pm_opp *suspend_opp;

	struct mutex genpd_virt_dev_lock;
	struct device **genpd_virt_devs;
	struct opp_table **required_opp_tables;
	unsigned int required_opp_count;

	unsigned int *supported_hw;
	unsigned int supported_hw_count;
	const char *prop_name;
	struct clk *clk;
	struct regulator **regulators;
	int regulator_count;
	struct icc_path **paths;
	unsigned int path_count;
	bool enabled;
	bool genpd_performance_state;
	bool is_genpd;

	int (*set_opp)(struct dev_pm_set_opp_data *data);
	struct dev_pm_set_opp_data *set_opp_data;

#ifdef CONFIG_DEBUG_FS
	struct dentry *dentry;
	char dentry_name[NAME_MAX];
#endif
};

// drivers/opp/core.c
/*
 * The root of the list of all opp-tables. All opp_table structures branch off
 * from here, with each opp_table containing the list of opps it supports in
 * various states of availability.
 */
// opp_tables是opp_table链表的头节点
LIST_HEAD(opp_tables);

static struct opp_device *_find_opp_dev(const struct device *dev,
							struct opp_table *opp_table)
{
	struct opp_device *opp_dev;
	// 从opp_table的dev_list中的dev与当前dev进行对比
	list_for_each_entry(opp_dev, &opp_table->dev_list, node)
		if (opp_dev->dev == dev)
			return opp_dev;

	return NULL;
}

static struct opp_table *_find_opp_table_unlocked(struct device *dev)
{
	struct opp_table *opp_table;
	bool found;
	// 在opp_tables中进行遍历查找当前设备对应的opp_table
	list_for_each_entry(opp_table, &opp_tables, node) {
		mutex_lock(&opp_table->lock);
		found = !!_find_opp_dev(dev, opp_table);
		mutex_unlock(&opp_table->lock);

		if (found) {
			// opp_table的引用计数加一
			_get_opp_table_kref(opp_table);

			return opp_table;
		}
	}

	return ERR_PTR(-ENODEV);
}

static struct opp_table *_opp_get_opp_table(struct device *dev, int index)
{
	struct opp_table *opp_table;

	/* Hold our table modification lock here */
	mutex_lock(&opp_table_lock);

	// 第一次运行到这里的时候，应该是找不到opp_table的，需要进行创建
	opp_table = _find_opp_table_unlocked(dev);
	if (!IS_ERR(opp_table))
		goto unlock;

	opp_table = _managed_opp(dev, index);
	if (opp_table) {
		if (!_add_opp_dev_unlocked(dev, opp_table)) {
			dev_pm_opp_put_opp_table(opp_table);
			opp_table = ERR_PTR(-ENOMEM);
		}
		goto unlock;
	}
	// 对该设备创建opp表
	opp_table = _allocate_opp_table(dev, index);

unlock:
	mutex_unlock(&opp_table_lock);

	return opp_table;
}


struct opp_table *dev_pm_opp_get_opp_table(struct device *dev)
{
	return _opp_get_opp_table(dev, 0);
}

int dev_pm_opp_add(struct device *dev, unsigned long freq, unsigned long u_volt)
{
	struct opp_table *opp_table;
	int ret;

	opp_table = dev_pm_opp_get_opp_table(dev);
	if (IS_ERR(opp_table)) 
		return PTR_ERR(opp_table);
	
	opp_table->regulator_count = 1;
	
	ret = _opp_add_v1(opp_table, dev, freq, u_volt, true);
	if (ret)
		dev_pm_opp_put_opp_table(opp_table);
	
	return ret;
}
```

最终获取得到的OPP表如下
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230222105334.png)

##### cpufreq初始化过程
cpufreq被注册cpu_subsys总线上

cpufreq的初始化从cpufreq_drvier注册开始，`cpufreq_register_driver()`函数为cpufreq驱动注册的入口，驱动程序通过调用该函数进行初始化，传入相关的`struct cpufreq_driver`，`cpufreq_register_driver()`会调用`subsys_interface_register()`最终执行回调函数`cpufreq_add_dev`，然后调用`cpufreq_online()`走初始化流程

```c
// cpufreq_drvier结构体
struct cpufreq_driver {
	char		name[CPUFREQ_NAME_LEN];
	u16		    flags;
	void		*driver_data;

	/* needed by all drivers */
	int		(*init)(struct cpufreq_policy *policy);
	int		(*verify)(struct cpufreq_policy_data *policy);
	// 调频接口
	int		(*target_index)(struct cpufreq_policy *policy,
					unsigned int index);
	unsigned int	(*fast_switch)(struct cpufreq_policy *policy,
				       unsigned int target_freq);

	/* should be defined, if possible */
	// 获取频率接口
	unsigned int	(*get)(unsigned int cpu);

	/* Called to update policy limits on firmware notifications. */
	void		(*update_limits)(unsigned int cpu);

	int		(*online)(struct cpufreq_policy *policy);
	int		(*offline)(struct cpufreq_policy *policy);
	int		(*exit)(struct cpufreq_policy *policy);

	struct freq_attr **attr;
};

// driver/base/cpu.c
// cpu subsys总线，cpufreq就是以subsys_interface挂在该总线下
struct bus_type cpu_subsys = {
	.name = "cpu",
	.dev_name = "cpu",
	.match = cpu_subsys_match,
	.online = cpu_subsys_online,
	.offline = cpu_subsys_offline,
};

// driver/cpufreq/cpufreq.c
// 指向当前使用的cpufreq_driver
static struct cpufreq_driver *cpufreq_driver;

// cpufreq subsys接口，用来挂到CPU subsys总线上
static struct subsys_interface cpufreq_interface = {
	.name		= "cpufreq",
	.subsys		= &cpu_subsys,
	.add_dev	= cpufreq_add_dev,
	.remove_dev	= cpufreq_remove_dev,
};

// scmi cpufreq_driver结构体定义
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

// 为每个cluster定义一个cpufreq_policy结构体，对每个cluster上的CPU进行调频管理
// 其中又分别进行cpufreq_driver的初始化和governor的初始化
static DEFINE_PER_CPU(struct cpufreq_policy *, cpufreq_cpu_data);

// 指向当前注册的cpufreq driver
static struct cpufreq_driver *cpufreq_driver;

// cpufreq驱动框架初始化过程，整个过程都围绕着policy这个结构体进行，逐步进行初始化
cpufreq_register_driver(&scmi_cpufreq_driver);
	subsys_interface_register(&cpufreq_interface);
		cpufreq_add_dev(dev, sif);
			cpufreq_online(cpu);
				// 初步初始化policy
				// cpumask初始化
				// policy->kobj kobject初始化
				// policy->constraints 频率限制初始化
				// 注册频率限制通知接口，频率最大最小值变化时会调用接口
				policy = cpufreq_policy_alloc(cpu);
				// 调用cpufreq_drvier init接口，完善policy结构体
				// 将opp表添加到对应的device，通过dev_pm_opp_add接口
				// 生成频率表 freq_table
				cpufreq_driver->init(policy) -> scmi_cpufreq_init(policy)
				cpufreq_table_validate_and_sort(policy);
				// 创建/sys/device/system/cpu/cpux目录下的cpufreq符号链接
				add_cpu_dev_symlink();
				freq_qos_and_request();
				blocking_notifier_call_chain();
				// CPU进行频率调整，使当前运行频率在频率表中
				// 初始化的时候会进行一次调整
				__cpufreq_driver_target();
				// 创建sys节点，/sys/device/system/cpu/cpufreq/policyx目录下的一些可选属性
				cpufreq_add_dev_interface(policy);
				cpufreq_stats_create_table(policy);
				list_add(&policy->polic_list, &cpufreq_policy_list);
				// 使用默认governor初始化policy
				cpufreq_init_policy();
```

来看一下`subsys_interface_register()`
```c
// drivers/base/bus.c
int subsys_interface_register(struct subsystem_interface *sif)
{
	struct bus_type *subsys;
	struct subsys_dev_iter iter;
	struct device *dev;

	if (!sif || !sif->subsys)
		return -ENODEV;

	subsys = bus_get(sif->subsys);
	if (!subsys)
		return -EINVAL;

	mutex_lock(&subsys->p->mutex);
	// 将cpufreq_interface添加到cpu_subsys总线的interfaces上
	// interfaces是一个list_head链表
	list_add_tail(&sif->node, &subsys->p->interfaces);

	// 遍历cpu_subsys总线
	// subsys_dev_iter是对klist 迭代器的一个封装
	// 这里遍历的对象是struct device私有数据的knode_bus
	// knode_bus节点是挂载到klist_device这个链表上
	if (sif->add_dev) {
		subsys_dev_iter_init(&iter, subsys, NULL, NULL);
		// 从knode_bus取得device指针
		while ((dev = subsys_dev_iter_next(&iter)))
			sif->add_dev(dev, sif);
		subsys_dev_iter_exit(&iter);
	}
	mutex_unlock(&subsys->p->mutex);

	return 0;
}
```

再来看看cpufreq_online()
```c
static int cpufreq_online(unsigned int cpu)
{
	struct cpufreq_policy *policy;
	bool new_policy;
	unsigned long flags;
	unsigned int j;
	int ret;

	pr_debug("%s: bringing CPU%u online\n", __func__, cpu);

	/* Check if this CPU already has a policy to manage it */
	// cpufreq_cpu_data是类型为policy指针的precpu变量
	// 这是是取得policy指针
	policy = per_cpu(cpufreq_cpu_data, cpu);
	if (policy) {
		// 假如该cpu不在该policy的related_cpus里面则是有问题的
		WARN_ON(!cpumask_test_cpu(cpu, policy->related_cpus));
		// 判断当前policy还有没有online CPU
		if (!policy_is_inactive(policy))
			// 将当前cpu加入到policy->cpu online CPU里面
			return cpufreq_add_policy_cpu(policy, cpu);

		/* This is the only online CPU for the policy.  Start over. */
		new_policy = false;
		down_write(&policy->rwsem);
		policy->cpu = cpu;
		policy->governor = NULL;
		up_write(&policy->rwsem);
	} else {
		// 第一次开机的时候需要分配policy内存
		new_policy = true;
		policy = cpufreq_policy_alloc(cpu);
		if (!policy)
			return -ENOMEM;
	}

	// scmi调频驱动没有实现online接口
	if (!new_policy && cpufreq_driver->online) {
		ret = cpufreq_driver->online(policy);
		if (ret) {
			pr_debug("%s: %d: initialization failed\n", __func__,
				 __LINE__);
			goto out_exit_policy;
		}

		/* Recover policy->cpus using related_cpus */
		cpumask_copy(policy->cpus, policy->related_cpus);
	} else {
		cpumask_copy(policy->cpus, cpumask_of(cpu));

		/*
		 * Call driver. From then on the cpufreq must be able
		 * to accept all calls to ->verify and ->setpolicy for this CPU.
		 */
		// cpufreq_driver是在cpufreq_register_driver()中进行赋值的
		// 调频驱动初始化
		ret = cpufreq_driver->init(policy);
		if (ret) {
			pr_debug("%s: %d: initialization failed\n", __func__,
				 __LINE__);
			goto out_free_policy;
		}

		ret = cpufreq_table_validate_and_sort(policy);
		if (ret)
			goto out_exit_policy;

		/* related_cpus should at least include policy->cpus. */
		cpumask_copy(policy->related_cpus, policy->cpus);
	}

	down_write(&policy->rwsem);
	/*
	 * affected cpus must always be the one, which are online. We aren't
	 * managing offline cpus here.
	 */
	cpumask_and(policy->cpus, policy->cpus, cpu_online_mask);

	if (new_policy) {
		for_each_cpu(j, policy->related_cpus) {
			per_cpu(cpufreq_cpu_data, j) = policy;
			add_cpu_dev_symlink(policy, j);
		}

		policy->min_freq_req = kzalloc(2 * sizeof(*policy->min_freq_req),
					       GFP_KERNEL);
		if (!policy->min_freq_req)
			goto out_destroy_policy;

		ret = freq_qos_add_request(&policy->constraints,
					   policy->min_freq_req, FREQ_QOS_MIN,
					   policy->min);
		if (ret < 0) {
			/*
			 * So we don't call freq_qos_remove_request() for an
			 * uninitialized request.
			 */
			kfree(policy->min_freq_req);
			policy->min_freq_req = NULL;
			goto out_destroy_policy;
		}

		/*
		 * This must be initialized right here to avoid calling
		 * freq_qos_remove_request() on uninitialized request in case
		 * of errors.
		 */
		policy->max_freq_req = policy->min_freq_req + 1;

		ret = freq_qos_add_request(&policy->constraints,
					   policy->max_freq_req, FREQ_QOS_MAX,
					   policy->max);
		if (ret < 0) {
			policy->max_freq_req = NULL;
			goto out_destroy_policy;
		}

		blocking_notifier_call_chain(&cpufreq_policy_notifier_list,
				CPUFREQ_CREATE_POLICY, policy);
	}

	if (cpufreq_driver->get && has_target()) {
		policy->cur = cpufreq_driver->get(policy->cpu);
		if (!policy->cur) {
			pr_err("%s: ->get() failed\n", __func__);
			goto out_destroy_policy;
		}
	}

	/*
	 * Sometimes boot loaders set CPU frequency to a value outside of
	 * frequency table present with cpufreq core. In such cases CPU might be
	 * unstable if it has to run on that frequency for long duration of time
	 * and so its better to set it to a frequency which is specified in
	 * freq-table. This also makes cpufreq stats inconsistent as
	 * cpufreq-stats would fail to register because current frequency of CPU
	 * isn't found in freq-table.
	 *
	 * Because we don't want this change to effect boot process badly, we go
	 * for the next freq which is >= policy->cur ('cur' must be set by now,
	 * otherwise we will end up setting freq to lowest of the table as 'cur'
	 * is initialized to zero).
	 *
	 * We are passing target-freq as "policy->cur - 1" otherwise
	 * __cpufreq_driver_target() would simply fail, as policy->cur will be
	 * equal to target-freq.
	 */
	if ((cpufreq_driver->flags & CPUFREQ_NEED_INITIAL_FREQ_CHECK)
	    && has_target()) {
		unsigned int old_freq = policy->cur;

		/* Are we running at unknown frequency ? */
		ret = cpufreq_frequency_table_get_index(policy, old_freq);
		if (ret == -EINVAL) {
			ret = __cpufreq_driver_target(policy, old_freq - 1,
						      CPUFREQ_RELATION_L);

			/*
			 * Reaching here after boot in a few seconds may not
			 * mean that system will remain stable at "unknown"
			 * frequency for longer duration. Hence, a BUG_ON().
			 */
			BUG_ON(ret);
			pr_info("%s: CPU%d: Running at unlisted initial frequency: %u KHz, changing to: %u KHz\n",
				__func__, policy->cpu, old_freq, policy->cur);
		}
	}

	if (new_policy) {
		ret = cpufreq_add_dev_interface(policy);
		if (ret)
			goto out_destroy_policy;

		cpufreq_stats_create_table(policy);

		write_lock_irqsave(&cpufreq_driver_lock, flags);
		list_add(&policy->policy_list, &cpufreq_policy_list);
		write_unlock_irqrestore(&cpufreq_driver_lock, flags);
	}

	ret = cpufreq_init_policy(policy);
	if (ret) {
		pr_err("%s: Failed to initialize policy for cpu: %d (%d)\n",
		       __func__, cpu, ret);
		goto out_destroy_policy;
	}

	up_write(&policy->rwsem);

	kobject_uevent(&policy->kobj, KOBJ_ADD);

	/* Callback for handling stuff after policy is ready */
	if (cpufreq_driver->ready)
		cpufreq_driver->ready(policy);

	if (cpufreq_thermal_control_enabled(cpufreq_driver))
		policy->cdev = of_cpufreq_cooling_register(policy);

	pr_debug("initialization complete\n");

	return 0;

out_destroy_policy:
	for_each_cpu(j, policy->real_cpus)
		remove_cpu_dev_symlink(policy, get_cpu_device(j));

	up_write(&policy->rwsem);

out_exit_policy:
	if (cpufreq_driver->exit)
		cpufreq_driver->exit(policy);

out_free_policy:
	cpufreq_policy_free(policy);
	return ret;
}
```

##### cpufreq drviver初始化
在cpufreq_online()中调用全局变量cpufreq_driver->init(policy)进行调频驱动的初始化，下面是scmi调频驱动的初始化过程
```c
static int scmi_cpufreq_init(struct cpufreq_policy *policy)
{
	int ret, nr_opp;
	unsigned int latency;
	struct device *cpu_dev;
	struct scmi_data *priv;
	struct cpufreq_frequency_table *freq_table;
	struct em_data_callback em_cb = EM_DATA_CB(scmi_get_cpu_power);

	// 获取CPU device结构体即对应percpu变量cpu_sys_devices
	cpu_dev = get_cpu_device(policy->cpu);
	if (!cpu_dev) {
		pr_err("failed to get cpu%d device\n", policy->cpu);
		return -ENODEV;
	}

	// 调用scmi_dvfs_device_opps_add()生成opp表
	ret = handle->perf_ops->device_opps_add(handle, cpu_dev);
	if (ret) {
		dev_warn(cpu_dev, "failed to add opps to the device\n");
		return ret;
	}
	// 遍历可用的cpu，对比cpu_dev的domain_id，若是相同的话加入到policy->cpus里面
	ret = scmi_get_sharing_cpus(cpu_dev, policy->cpus);
	if (ret) {
		dev_warn(cpu_dev, "failed to get sharing cpumask\n");
		return ret;
	}
	// 将opp表添加到sharing cpus里面
	ret = dev_pm_opp_set_sharing_cpus(cpu_dev, policy->cpus);
	if (ret) {
		dev_err(cpu_dev, "%s: failed to mark OPPs as shared: %d\n",
			__func__, ret);
		return ret;
	}

	nr_opp = dev_pm_opp_get_opp_count(cpu_dev);
	if (nr_opp <= 0) {
		dev_dbg(cpu_dev, "OPP table is not ready, deferring probe\n");
		ret = -EPROBE_DEFER;
		goto out_free_opp;
	}

	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv) {
		ret = -ENOMEM;
		goto out_free_opp;
	}
	// cpufreq
	ret = dev_pm_opp_init_cpufreq_table(cpu_dev, &freq_table);
	if (ret) {
		dev_err(cpu_dev, "failed to init cpufreq table: %d\n", ret);
		goto out_free_priv;
	}

	priv->cpu_dev = cpu_dev;
	priv->domain_id = handle->perf_ops->device_domain_id(cpu_dev);
	// driver_data保存了调频驱动的一些数据，这里只有cpu编号和domain id号
	policy->driver_data = priv;
	policy->freq_table = freq_table;

	/* SCMI allows DVFS request for any domain from any CPU */
	policy->dvfs_possible_from_any_cpu = true;

	latency = handle->perf_ops->transition_latency_get(handle, cpu_dev);
	if (!latency)
		latency = CPUFREQ_ETERNAL;

	policy->cpuinfo.transition_latency = latency;

	policy->fast_switch_possible =
		handle->perf_ops->fast_switch_possible(handle, cpu_dev);

	em_dev_register_perf_domain(cpu_dev, nr_opp, &em_cb, policy->cpus);

	return 0;

out_free_priv:
	kfree(priv);
out_free_opp:
	dev_pm_opp_remove_all_dynamic(cpu_dev);

	return ret;
}
```


频率表初始化过程
```c
struct cpufreq_frequency_table {
	unsigned int flags;
	unsigned int driver_data;
	unsigned int frequency;		/* KHz */
};

// drivers/opp/cpu.c
int dev_pm_opp_init_cpufreq_table(struct device *dev,
				  struct cpufreq_frequency_table **table)
{
	struct dev_pm_opp *opp;
	struct cpufreq_frequency_table *freq_table = NULL;
	int i, max_opps, ret = 0;
	unsigned long rate;

	max_opps = dev_pm_opp_get_opp_count(dev);
	if (max_opps <= 0)
		return max_opps ? max_opps : -ENODATA;

	freq_table = kcalloc((max_opps + 1), sizeof(*freq_table), GFP_KERNEL);
	if (!freq_table)
		return -ENOMEM;

	for (i = 0, rate = 0; i < max_opps; i++, rate++) {
		/* find next rate */
		opp = dev_pm_opp_find_freq_ceil(dev, &rate);
		if (IS_ERR(opp)) {
			ret = PTR_ERR(opp);
			goto out;
		}
		freq_table[i].driver_data = i;
		freq_table[i].frequency = rate / 1000;

		/* Is Boost/turbo opp ? */
		if (dev_pm_opp_is_turbo(opp))
			freq_table[i].flags = CPUFREQ_BOOST_FREQ;

		dev_pm_opp_put(opp);
	}

	freq_table[i].driver_data = i;
	freq_table[i].frequency = CPUFREQ_TABLE_END;

	*table = &freq_table[0];

out:
	if (ret)
		kfree(freq_table);

	return ret;
}
```


##### governor初始化过程
cpufreq governor的初始化过程，在cpufreq_init_policy(policy)中进行，这里以ondemand为例进行分析
```c
// include/linux/cpufreq.h
// 内核governor描述结构体，形成链表
struct cpufreq_governor {
	char	name[CPUFREQ_NAME_LEN];
	int	(*init)(struct cpufreq_policy *policy);
	void	(*exit)(struct cpufreq_policy *policy);
	int	(*start)(struct cpufreq_policy *policy);
	void	(*stop)(struct cpufreq_policy *policy);
	void	(*limits)(struct cpufreq_policy *policy);
	ssize_t	(*show_setspeed)	(struct cpufreq_policy *policy,
					 char *buf);
	int	(*store_setspeed)	(struct cpufreq_policy *policy,
					 unsigned int freq);
	struct list_head	governor_list;
	struct module		*owner;
	u8			flags;
};

/* Common Governor data across policies */
// 抽象出的ondemand governor结构体
// drivers/cpufreq/cpufreq_governor.h
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

/* Governor demand based switching data (per-policy or global). */
// ondemand计算频率使用的相关参数，包括阈值 采样率等，默认阈值是负载的80%
// dbs(demand based switching)按需切换
struct dbs_data {
	struct gov_attr_set attr_set;
	void *tuners;
	unsigned int ignore_nice_load;
	unsigned int sampling_rate;
	unsigned int sampling_down_factor;
	unsigned int up_threshold;
	unsigned int io_is_busy;
};

/* Common to all CPUs of a policy */
// driver/cpufreq/cpufreq_governor.h
struct policy_dbs_info {
	struct cpufreq_policy *policy;
	/*
	 * Per policy mutex that serializes load evaluation from limit-change
	 * and work-handler.
	 */
	struct mutex update_mutex;

	u64 last_sample_time;
	s64 sample_delay_ns;
	atomic_t work_count;
	struct irq_work irq_work;
	struct work_struct work;
	/* dbs_data may be shared between multiple policy objects */
	struct dbs_data *dbs_data;
	struct list_head list;
	/* Multiplier for increasing sample delay temporarily. */
	unsigned int rate_mult;
	unsigned int idle_periods;	/* For conservative */
	/* Status indicators */
	bool is_shared;		/* This object is used by multiple CPUs */
	bool work_in_progress;	/* Work is being queued up or in progress */
};

// drivers/cpufreq/cpufreq_governor.h
// governor初始化宏
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

// ondemand governor定义
// driver/cpufreq/cpufreq_ondemand.c
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

// 初始化governor
// 该函数会在governor模块驱动的入口函数调用
// 只要编译该模块，就会注册到cpufreq framework中
#define CPU_FREQ_GOV_ONDEMAND	(od_dbs_gov.gov)
cpufreq_governor_init(CPU_FREQ_GOV_ONDEMAND);

#define cpufreq_governor_init(__governor)			\
static int __init __governor##_init(void)			\
{								\
	return cpufreq_register_governor(&__governor);	\
}								\
core_initcall(__governor##_init)

// 在cpufreq_online()中调用默认governor对policy进行完善，启动当前governor
cpufreq_init_policy(policy);
	gov = get_governor(default_governor);
	// 设置新的governor
	cpufreq_set_policy(policy, gov, pol);
		cpufreq_init_governor(policy);
			policy->governor->init();  -> cpufreq_dbs_governor_init(policy);
				gov->init(dbs_data); -> odinit(dbs_data);
		cpufreq_start_governor(policy);
			policy->governor->start(); -> cpufreq_dbs_governor_start(policy);
				// 设置governor回调函数， 以ondemand为例
				gov_set_update_util(policy_dbs, sampling_rate);
					cpufreq_add_update_util_hook(cpu, &cdbs->updata_util,
									dbs_update_util_handler);
```
启动governor中比较重要的是设置调频回调函数,该函数是真正调频时计算合适频率的函数

#### ondemand调节器

ondemand调节器也会根据当前的CPU负载来进行CPU频率计算，ondemand工作过程如下

```c
// include/linux/sched/cpufreq.h
// CPU利用率变化时会调用func重新计算频率
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};

// drivers/cpufreq/cpufreq_governor.c
// 设置governor回调函数update_util_data->func = dbs_update_util_handler
cpufreq_dbs_governor_start()
    gov_set_update_util(policy_dbs, sampling_rate);
		cpufreq_add_update_util_hook(cpu, &cdbs->update_util,
                                    dbs_update_util_handler);

/**
 * cpufreq_update_util - Take a note about CPU utilization changes.
 * @rq: Runqueue to carry out the update for.
 * @flags: Update reason flags.
 *
 * This function is called by the scheduler on the CPU whose utilization is
 * being updated.
 *
 * It can only be called from RCU-sched read-side critical sections.
 *
 * The way cpufreq is currently arranged requires it to evaluate the CPU
 * performance state (frequency/voltage) on a regular basis to prevent it from
 * being stuck in a completely inadequate performance level for too long.
 * That is not guaranteed to happen if the updates are only triggered from CFS
 * and DL, though, because they may not be coming in if only RT tasks are
 * active all the time (or there are RT tasks only).
 *
 * As a workaround for that issue, this function is called periodically by the
 * RT sched class to trigger extra cpufreq updates to prevent it from stalling,
 * but that really is a band-aid.  Going forward it should be replaced with
 * solutions targeted more specifically at RT tasks.
 */
// kernel/sched/sched.h
// 当cpufreq_update_util()被调用时执行上面设置的回调函数update_util_data->func
// 为了考虑RT任务的影响，目前在RT调度中会周期性调用该函数，避免CPU频率更新不及时
static inline void cpufreq_update_util(struct rq *rq, unsigned int flags)
{
    struct update_util_data *data;
    data = rcu_dereference_sched(*per_cpu_ptr(&cpufreq_update_util_data,
                                cpu_of(rq)));
    if (data)
        data->func(data, rq_clock(rq), flags);
}

// drivers/cpufreq/cpufreq_governor.c
// data: 为了后续计算过程能够使用container_of取到cpu_dbs_info地址
// time: 运行队列更新的时间
// flags: 更新的原因标志
static void dbs_update_util_handler(struct update_util_data *data, u64 time,
				    unsigned int flags)
{
	struct cpu_dbs_info *cdbs = container_of(data, struct cpu_dbs_info, update_util);
	struct policy_dbs_info *policy_dbs = cdbs->policy_dbs;
	u64 delta_ns, lst;

    // 检查当前工作的CPU可否对该policy对应的CPU进行调频，E2000是支持的
	if (!cpufreq_this_cpu_can_update(policy_dbs->policy))
		return;

	/*
	 * The work may not be allowed to be queued up right now.
	 * Possible reasons:
	 * - Work has already been queued up or is in progress.
	 * - It is too early (too little time from the previous sample).
	 */
	if (policy_dbs->work_in_progress)
		return;

	/*
	 * If the reads below are reordered before the check above, the value
	 * of sample_delay_ns used in the computation may be stale.
	 */
    // 判断更新的时间间隔，假如小于smaple_delay_ns直接返回
	smp_rmb();
	lst = READ_ONCE(policy_dbs->last_sample_time);
	delta_ns = time - lst;
	if ((s64)delta_ns < policy_dbs->sample_delay_ns)
		return;

	/*
	 * If the policy is not shared, the irq_work may be queued up right away
	 * at this point.  Otherwise, we need to ensure that only one of the
	 * CPUs sharing the policy will do that.
	 */
    // 若policy是被多个CPU共享的，那么要判断一下是不是已经有CPU对其进行处理过了
	if (policy_dbs->is_shared) {
		if (!atomic_add_unless(&policy_dbs->work_count, 1, 1))
			return;

		/*
		 * If another CPU updated last_sample_time in the meantime, we
		 * shouldn't be here, so clear the work counter and bail out.
		 */
		if (unlikely(lst != READ_ONCE(policy_dbs->last_sample_time))) {
			atomic_set(&policy_dbs->work_count, 0);
			return;
		}
	}

    // 更新标志位
	policy_dbs->last_sample_time = time;
	policy_dbs->work_in_progress = true;
	irq_work_queue(&policy_dbs->irq_work);
}

// drivers/cpufreq/cpufreq_governor.c
// init_irq_work(&policy_dbs->irq_work, dbs_irq_work)
static void dbs_irq_work(struct irq_work *irq_work)
{
	struct policy_dbs_info *policy_dbs;

	policy_dbs = container_of(irq_work, struct policy_dbs_info, irq_work);
    // 调度policy_dbs->work执行
	schedule_work_on(smp_processor_id(), &policy_dbs->work);
}

// drivers/cpufreq/cpufreq_governor.c
// INIT_WORK(&policy_dbs->work, dbs_work_handler)
static void dbs_work_handler(struct work_struct *work)
{
	struct policy_dbs_info *policy_dbs;
	struct cpufreq_policy *policy;
	struct dbs_governor *gov;

	policy_dbs = container_of(work, struct policy_dbs_info, work);
	policy = policy_dbs->policy;
	gov = dbs_governor_of(policy);

	/*
	 * Make sure cpufreq_governor_limits() isn't evaluating load or the
	 * ondemand governor isn't updating the sampling rate in parallel.
	 */
    // 调用gov->gov_dbs_update()接口
	mutex_lock(&policy_dbs->update_mutex);
	gov_update_sample_delay(policy_dbs, gov->gov_dbs_update(policy));
	mutex_unlock(&policy_dbs->update_mutex);

	/* Allow the utilization update handler to queue up more work. */
	atomic_set(&policy_dbs->work_count, 0);
	/*
	 * If the update below is reordered with respect to the sample delay
	 * modification, the utilization update handler may end up using a stale
	 * sample delay value.
	 */
	smp_wmb();
	policy_dbs->work_in_progress = false;
}

// drivers/cpufreq/cpufreq_ondemand.c
// ondemand回调函数，按照CPU负载计算频率
static unsigned int od_dbs_update(struct cpufreq_policy *policy)
{
	struct policy_dbs_info *policy_dbs = policy->governor_data;
	struct dbs_data *dbs_data = policy_dbs->dbs_data;
	struct od_policy_dbs_info *dbs_info = to_dbs_info(policy_dbs);
	int sample_type = dbs_info->sample_type;

	/* Common NORMAL_SAMPLE setup */
	dbs_info->sample_type = OD_NORMAL_SAMPLE;
	/*
	 * OD_SUB_SAMPLE doesn't make sense if sample_delay_ns is 0, so ignore
	 * it then.
	 */
	if (sample_type == OD_SUB_SAMPLE && policy_dbs->sample_delay_ns > 0) {
		__cpufreq_driver_target(policy, dbs_info->freq_lo,
					CPUFREQ_RELATION_H);
		return dbs_info->freq_lo_delay_us;
	}

	od_update(policy);

	if (dbs_info->freq_lo) {
		/* Setup SUB_SAMPLE */
		dbs_info->sample_type = OD_SUB_SAMPLE;
		return dbs_info->freq_hi_delay_us;
	}

	return dbs_data->sampling_rate * policy_dbs->rate_mult;
}

/*
 * Every sampling_rate, we check, if current idle time is less than 20%
 * (default), then we try to increase frequency. Else, we adjust the frequency
 * proportional to load.
 */
// drivers/cpufreq/cpufreq_ondemand.c
// 根据CPU负载，调整频率
static void od_update(struct cpufreq_policy *policy)
{
	struct policy_dbs_info *policy_dbs = policy->governor_data;
	struct od_policy_dbs_info *dbs_info = to_dbs_info(policy_dbs);
	struct dbs_data *dbs_data = policy_dbs->dbs_data;
	struct od_dbs_tuners *od_tuners = dbs_data->tuners;
	unsigned int load = dbs_update(policy);

	dbs_info->freq_lo = 0;

	/* Check for frequency increase */
    // 检查当前的CPU负载，负载大于80%（也可以在sysfs中设置）
	if (load > dbs_data->up_threshold) {
		/* If switching to max speed, apply sampling_down_factor */
		if (policy->cur < policy->max)
			policy_dbs->rate_mult = dbs_data->sampling_down_factor;
		dbs_freq_increase(policy, policy->max);
	} else {
		/* Calculate the next frequency proportional to load */
		unsigned int freq_next, min_f, max_f;

		min_f = policy->cpuinfo.min_freq;
		max_f = policy->cpuinfo.max_freq;
        // 按照CPU负载计算调频频率
		freq_next = min_f + load * (max_f - min_f) / 100;

		/* No longer fully busy, reset rate_mult */
		policy_dbs->rate_mult = 1;

        // 为了进一步节省电力，在计算出的新频率上再乘以一个powersave_bias设定的百分比
        // powersave_bias的值从0-1000，每一步表示0.1%
		if (od_tuners->powersave_bias)
			freq_next = od_ops.powersave_bias_target(policy,
								 freq_next,
								 CPUFREQ_RELATION_L);
		// 频率调整
		__cpufreq_driver_target(policy, freq_next, CPUFREQ_RELATION_C);
	}
}

// drivers/cpufreq/cpufreq_governor.c
// 计算当前域CPU负载 cpu_load = 100 * (time_elapsed - idle_time) / time_elapsed
// 返回当前域中最大的CPU负载
unsigned int dbs_update(struct cpufreq_policy *policy)
{
	struct policy_dbs_info *policy_dbs = policy->governor_data;
	struct dbs_data *dbs_data = policy_dbs->dbs_data;
	unsigned int ignore_nice = dbs_data->ignore_nice_load;
	unsigned int max_load = 0, idle_periods = UINT_MAX;
	unsigned int sampling_rate, io_busy, j;

	/*
	 * Sometimes governors may use an additional multiplier to increase
	 * sample delays temporarily.  Apply that multiplier to sampling_rate
	 * so as to keep the wake-up-from-idle detection logic a bit
	 * conservative.
	 */
	sampling_rate = dbs_data->sampling_rate * policy_dbs->rate_mult;
	/*
	 * For the purpose of ondemand, waiting for disk IO is an indication
	 * that you're performance critical, and not that the system is actually
	 * idle, so do not add the iowait time to the CPU idle time then.
	 */
	io_busy = dbs_data->io_is_busy;

	/* Get Absolute Load */
    // 计算当前policy中CPU的最大负载
	for_each_cpu(j, policy->cpus) {
		struct cpu_dbs_info *j_cdbs = &per_cpu(cpu_dbs, j);
		u64 update_time, cur_idle_time;
		unsigned int idle_time, time_elapsed;
		unsigned int load;

        // 获取当前idle时间
		cur_idle_time = get_cpu_idle_time(j, &update_time, io_busy);

        // time_elapsed = 本次总运行时间 - 上次总运行时间
		time_elapsed = update_time - j_cdbs->prev_update_time;
		j_cdbs->prev_update_time = update_time;

		// 计算本次时间间隔内的idle_time
        // idle_time = 本次idle时间 - 上次idle时间
		idle_time = cur_idle_time - j_cdbs->prev_cpu_idle;
		j_cdbs->prev_cpu_idle = cur_idle_time;

		if (ignore_nice) {
			u64 cur_nice = kcpustat_field(&kcpustat_cpu(j), CPUTIME_NICE, j);

			idle_time += div_u64(cur_nice - j_cdbs->prev_cpu_nice, NSEC_PER_USEC);
			j_cdbs->prev_cpu_nice = cur_nice;
		}

		// 这里主要是对各种比较罕见的情况进行临时处理
		if (unlikely(!time_elapsed)) {
			/*
			 * That can only happen when this function is called
			 * twice in a row with a very short interval between the
			 * calls, so the previous load value can be used then.
			 */
			load = j_cdbs->prev_load;
		} else if (unlikely((int)idle_time > 2 * sampling_rate &&
				    j_cdbs->prev_load)) {
			/*
			 * If the CPU had gone completely idle and a task has
			 * just woken up on this CPU now, it would be unfair to
			 * calculate 'load' the usual way for this elapsed
			 * time-window, because it would show near-zero load,
			 * irrespective of how CPU intensive that task actually
			 * was. This is undesirable for latency-sensitive bursty
			 * workloads.
			 *
			 * To avoid this, reuse the 'load' from the previous
			 * time-window and give this task a chance to start with
			 * a reasonably high CPU frequency. However, that
			 * shouldn't be over-done, lest we get stuck at a high
			 * load (high frequency) for too long, even when the
			 * current system load has actually dropped down, so
			 * clear prev_load to guarantee that the load will be
			 * computed again next time.
			 *
			 * Detecting this situation is easy: an unusually large
			 * 'idle_time' (as compared to the sampling rate)
			 * indicates this scenario.
			 */
			load = j_cdbs->prev_load;
			j_cdbs->prev_load = 0;
		// 程序正常会运行到这里进行负载计算
		} else {
			if (time_elapsed >= idle_time) {
				load = 100 * (time_elapsed - idle_time) / time_elapsed;
			} else {
				/*
				 * That can happen if idle_time is returned by
				 * get_cpu_idle_time_jiffy().  In that case
				 * idle_time is roughly equal to the difference
				 * between time_elapsed and "busy time" obtained
				 * from CPU statistics.  Then, the "busy time"
				 * can end up being greater than time_elapsed
				 * (for example, if jiffies_64 and the CPU
				 * statistics are updated by different CPUs),
				 * so idle_time may in fact be negative.  That
				 * means, though, that the CPU was busy all
				 * the time (on the rough average) during the
				 * last sampling interval and 100 can be
				 * returned as the load.
				 */
				load = (int)idle_time < 0 ? 100 : 0;
			}
			j_cdbs->prev_load = load;
		}

		if (unlikely((int)idle_time > 2 * sampling_rate)) {
			unsigned int periods = idle_time / sampling_rate;

			if (periods < idle_periods)
				idle_periods = periods;
		}
		// 取当前policy中CPU负载最大的值
		if (load > max_load)
			max_load = load;
	}

	policy_dbs->idle_periods = idle_periods;

	return max_load;
}
```


#### schedutil调节器
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230103174117.png)
sugov（schedutil governor）作为一种内核调频策略模块，它主要是根据当前CPU的利用率进行调频。因此，sugov会注册一个callback函数（sugov_update_shared/sugov_update_single)到调度器负载跟踪模块，当CPU util发生变化的时候就会调用该callback函数，检查一下当前CPU频率是否和当前的CPU util匹配，如果不匹配，那么就进行升频或者降频。

`sugov_tunables`结构体，用来描述sugov的可调参数

```c
// sugov_tunables结构体
struct sugov_tunables {
	struct gov_attr_set attr_set;
	// 目前sugov只有这一个可调参数，该参数用来限制连续调频的间隔时间
	unsigned int rate_limit_us;
};
```

`sugov_policy`结构体，sugov为每个cluster构建了该数据结构，记录每个cluster的调频数据信息
```c
// sugov_policy结构体，为每个簇构建了该数据结构，记录每个簇的调频数据信息
struct sugov_policy {
	// 指向cpufreq framework层的policy对象
	struct cpufreq_policy	*policy;
	// sugov的可调参数
	struct sugov_tunables	*tunables;
	struct list_head	tunables_hook;

	raw_spinlock_t		update_lock;	/* For shared policies */
	// 记录上次进行频率调整的时间点
	u64			last_freq_update_time;
	// 最小调频时间间隔
	s64			freq_update_delay_ns;
	// 下一个需要调整到的频率值，回调函数主要是计算这个参数
	unsigned int		next_freq;
	// 根据CPU util计算出来的原始频率，在频率表中向上找最接近的频率进行调整
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

`sugov_cpu`结构体，sugov为每个cpu构建了该数据结构，记录每个CPU的调频数据信息

```c
struct sugov_cpu {
	// 保存了cpu util变化后的回调函数
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

sugov初始化过程和ondemand初始化过程相似，当内核设定默认governor为sugov时，在`cpufreq_init_governor(policy);`中会调用`sugov_init()`初始化sugov，然后调用`sugov_start()`设置调频回调函数，每当CPU利用率发生变化的时候，调度器都会调用`cpufreq_update_util()`通知sugov，在`cpufreq_update_util()`被调用时，即任务调度后CPU当前的util发生变化，会调用sugov的回调函数进行调频，`sugov_update_shared()`当一个簇中有多个CPU调用该回调，遍历簇上的CPU找到当前最大util的CPU，然后根据该util映射到频率；`sugov_update_single()`即一个簇上单个CPU的情况直接根据该CPU util计算频率

调度事件的发生还是非常密集的，特别是在重载的情况下，很多任务可能执行若干个us就切换出去了。如果每次都计算CPU util看看是否需要调整频率，那么本身sugov就给系统带来较重的负荷，因此并非每次调频时机都会真正执行调频检查，sugov设置了一个最小调频间隔，小于这个间隔的调频请求会被过滤掉。

##### schedutil频率计算过程
```c
// sugov_start会遍历该sugov policy（cluster）中的所有cpu
// 调用cpufreq_add_update_util_hook为sugov cpu注册调频回调函数，代码逻辑如下：
static int sugov_start(struct cpufreq_policy *policy)
{
	...
	for_each_cpu(cpu, policy->cpus) {
		struct sugov_cpu *sg_cpu = &per_cpu(sugov_cpu, cpu);
		// 设置governor 计算回调函数，cpufreq_update_util()被调用时
		// 即任务调度后CPU当前的util发生变化，会调用sugov的回调函数进行调频计算
		cpufreq_add_update_util_hook(cpu, &sg_cpu->update_util,
					     policy_is_shared(policy) ?
							sugov_update_shared :
							sugov_update_single);
	}
	...
}

// schedutil频率计算过程
sugov_update_single();
	// 调频最小间隔时间检查，小于设定时间，直接返回
	sugov_should_update_freq();
	util = sugov_get_util(sg_cpuu);
		schedutil_cpu_util(sg_cpu->cpu, util, max, FREQUENCY_UTIL, NULL);
	// 根据当前CPU的util映射到具体的频率上
	next_f = get_next_freq(sg_policy, util, max);
	// 调用cpufreq_driver进行调频
	sugov_deferred_update(sg_policy, time, next_f);
		__cpufreq_driver_target()

// 计算cpu当前的utility
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

	// 如果CPU处理了过多的中断服务函数，irq负载已经高过CPU最大算力，直接返回最大算力
	irq = cpu_util_irq(rq);
	if (unlikely(irq >= max))
		return max;

	// 累加了cfs和rt任务的utility
	util = util_cfs + cpu_util_rt(rq);
	if (type == FREQUENCY_UTIL)
		util = uclamp_rq_util_with(rq, util, p);

	dl_util = cpu_util_dl(rq);

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
	// irq会偷走一部分的cpu算力，从而让其capacity没有那么大。
	// 这里通过scale_irq_capacity对任务的utility进行调整
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

// 根据当前CPU计算的util映射对应频率
static unsigned int get_next_freq(struct sugov_policy *sg_policy,
				  unsigned long util, unsigned long max)
{
	struct cpufreq_policy *policy = sg_policy->policy;
	// 先取得当前CPU的最大频率
	unsigned int freq = arch_scale_freq_invariant() ?
				policy->cpuinfo.max_freq : policy->cur;
	// 计算当前util对应频率，计算公式: freq = (1.25) * freq * util / max
	// 这里冗余了25%的算力余量
	freq = map_util_freq(util, freq, max);

	// 若计算出的freq和上次缓存的一样，则实际调整的next_freq计算后肯定也是一样的，直接返回
	// 上次记录的频率值
	if (freq == sg_policy->cached_raw_freq && !sg_policy->need_freq_update)
		return sg_policy->next_freq;

	sg_policy->cached_raw_freq = freq;

	// 根据当前算的freq，在CPU频率表上查找对应的频率
	freq = cpufreq_driver_resolve_freq(policy, freq);

	return freq;
}
```

### EAS能源感知调度

#### EAS整体框架
![](https://raw.githubusercontent.com/JackHuang021/images/master/20221216104706.png)

完全公平调度（Completely Fair Scheduler CFS）实现了面向吞吐量的的任务调度策略，EAS为这个调度器添加了一个基于能耗的调度策略，在优化CPU算力冗余的同时实现了节能，EAS在系统中、低度负载情况下工作，CFS在系统满负载情况下工作。

EAS在CPU调度领域，在为任务选核是起作用，目的是保证性能的情况下尽可能节省功耗，EAS涉及内核的几个子系统（任务调度、能源管理、CPU动态调频），EAS代码主要位于`kernel/sched/fair.c`，能源感知的任务调度需要调度器评估各个任务在CPU上运行带来的能耗影响

EAS全局控制开关`/proc/sys/kernel/sched_energy_aware`

#### CPU算力归一化过程

当前，Linux无法凭自身算出CPU算力，因此必须要有把这个信息传递给Linux的方式，它是从`capacity-dmips-mhz` CPU 设备树binding中衍生计算出来的

归一化CPU capacity，`topology_normalize_cpu_scale()`定义在`drivers/base/arch_topology()`，这个capacity在schedutil调度中被`sugov_get_util()`函数读取

`topology_normalize_cpu_scale()`在CPU初始化`parse_dt_topology()`中被调用，capacity归一化的前提条件是需要在设备树中CPU节点设置`capacity-dmips-mhz`属性，该属性表示不同CPU的计算能力，内核读取该属性设置CPU的`raw_capacity`为`capacity-dmips-mhz`，参考内核文档`Documentation/devicetree/bindings/arm/cpu-capacity.txt`

> ARM推荐的测试CPU的性能工具：Dhrystone 2.1以上版本，可以通过单核跑分成绩作为`capacity-dmips-mhz`属性的参考，DMIPS： Dhrystone Million Instructions executed Per Second，表示了在Dhrystone这样一种测试方法下的MIPS，Dhrystone是一种整数运算测试程序。MIPS/MHz，就是说每MHz频率能产生多大的MIPS，CPU性能通常由每秒百万指令（Millions of Instructions Per Second，MIPS）表示，设备树里表示为dmips/mhz

CPU算力归一化公式，并不是简单的将capacity-dmips-mhz归一化到capacity，CPU的频率也参与到了计算中
` capacity = (own(capacity-dmips-mhz) * own(max_freq)) / (max(capacity-dmips-mhz) * max(max_freq)) * 1024`

根据测试部测试的E2000QCPU单核性能数据，E2000Q的`capacity-dmips-mhz`属性值可以设置为如下，放大1000倍：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230105152415.png)

```c
	// 小核
	cpu_l0: cpu@0 {
		...
		capacity-dmips-mhz = <2850>;
		...
	};
	// 大核
	cpu_b0: cpu@0 {
		...
		capacity-dmips-mhz = <5660>;
		...
	};
```
实际经过CPU算力归一化到1024之后，对应的小核CPU算力为386，大核为1024

#### EAS代码相关结构体

perf_domain结构表示一个CPU性能域，perf_domain和cpufreq_policy是一一对应的，性能域之间形成链，链表头存放在root_domain中
```c
// kernel/sched/sched.h
// perf_comain 结构表示一个CPU性能域，perf_domain和cpufreq_policy是一一对应的
struct perf_domain {
	struct em_perf_domain *em_pd;
	struct perf_domain *next;
	struct rcu_head rcu;
};

// cpumask_var_t解析
// include/uapi/linux/kernel.h
#define __KERNEL_DIV_ROUND_UP(n, d) ((((n) + (d)) - 1) / (d))

// include/linux/kernel.h
#define DIV_ROUND_UP	__KERNEL_DIV_ROUND_UP

// include/linux/bits.h
#define BITS_PER_BYTE 	8

// include/linux/bitops.h
#define BITS_PER_TYPE(type) (sizeof(type) * BITS_PER_BYTE)
#define BITS_TO_LONGS(nr)	DIV_ROUND_UP(nr, BITS_PER_TYPE(long))

// include/linux/types.h
#define DECLARE_BITMAP(name, bits) \
	unsigned long name[BITS_TO_LONGS(bits)]

// include/linux/threads.h
#define CONFIG_NR_CPUS 256
#define NR_CPUS		CONFIG_NR_CPUS

// include/linux/cpumask.h
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
typedef struct cpumask cpumask_var_t[1];

// 展开cpumask
// 该结构体用来表示当前系统中CPU的状态，每个CPU对应其中一位
// 这里内核配置最大支持256个CPU，所以定义了一个长度为4的long数组
typedef struct cpumask {
	/*
	unsigned long bits[BITS_TO_LONGS(256)];
	unsigned long bits[DIV_ROUND_UP(256, 64)];
	*/
	unsigned long bits[4];
} cpumask_t;

// root_domain代表CPU性能域的一个总体数据结构，管理这些性能域
struct root_domain {
	atomic_t		refcount;
	atomic_t		rto_count;
	struct rcu_head		rcu;
	cpumask_var_t		span;
	cpumask_var_t		online;

	// 该root_domain是否处于overload状态
	int			overload;

	// 该root_domain是否处于overutilized状态
	int			overutilized;

	/*
	 * The bit corresponding to a CPU gets set here if such CPU has more
	 * than one runnable -deadline task (as it is below for RT tasks).
	 */
	cpumask_var_t		dlo_mask;
	atomic_t		dlo_count;
	struct dl_bw		dl_bw;
	struct cpudl		cpudl;

#ifdef HAVE_RT_PUSH_IPI
	/*
	 * For IPI pull requests, loop across the rto_mask.
	 */
	struct irq_work		rto_push_work;
	raw_spinlock_t		rto_lock;
	/* These are only updated and read within rto_lock */
	int			rto_loop;
	int			rto_cpu;
	/* These atomics are updated outside of a lock */
	atomic_t		rto_loop_next;
	atomic_t		rto_loop_start;
#endif
	/*
	 * The "RT overload" flag: it gets set if a CPU has more than
	 * one runnable RT task.
	 */
	cpumask_var_t		rto_mask;
	struct cpupri		cpupri;

	// 系统中算力最大的CPU的算力
	unsigned long		max_cpu_capacity;

	/*
	 * NULL-terminated list of performance domains intersecting with the
	 * CPUs of the rd. Protected by RCU.
	 */
	// perf_domain单链表的表头
	struct perf_domain __rcu *pd;
};

// include/linux/energy_model.h
struct em_perf_state {
	unsigned long frequency;	// CPU频点，单位KHz
	unsigned long power;		// 此频点下的功耗
	unsigned long cost;			// 此频点下的成本系数，等于 power * max_freq / freq
};

struct em_perf_domain {
	struct em_perf_state *table;	// CPU频点表
	int nr_perf_states;				// 频点表中元素的个数
	unsigned long cpus[];			// 此性能域中包括哪些CPU
};
```

#### perf_domain初始化
start_kernel() -> sched_init() -> init_defrootdomain()
```c
// kernel/sched/core.c
void __init sched_init(void)
{
	...
#ifdef CONFIG_SMP
	init_defrootdmain();
#endif
	...
}

// kernel/sched/topology.c
static bool build_perf_domains(const struct cpumask *cpu_map)
{
	// 这里nr_cpus即统计cpu_map里面当前CPU的个数
	int i, nr_pd = 0, nr_ps = 0, nr_cpus = cpumask_weight(cpu_map);
	struct perf_domain *pd = NULL, *tmp;
	int cpu = cpumask_first(cpu_map);
	struct root_domain *rd = cpu_rq(cpu)->rd;
	struct cpufreq_policy *policy;
	struct cpufreq_governor *gov;

	if (!sysctl_sched_energy_aware)
		goto free;

	/* EAS is enabled for asymmetric CPU capacity topologies. */
	if (!per_cpu(sd_asym_cpucapacity, cpu)) {
		if (sched_debug()) {
			pr_info("rd %*pbl: CPUs do not have asymmetric capacities\n",
					cpumask_pr_args(cpu_map));
		}
		goto free;
	}

	/* EAS definitely does *not* handle SMT */
	if (sched_smt_active()) {
		pr_warn("rd %*pbl: Disabling EAS, SMT is not supported\n",
			cpumask_pr_args(cpu_map));
		goto free;
	}

	for_each_cpu(i, cpu_map) {
		/* Skip already covered CPUs. */
		if (find_pd(pd, i))
			continue;

		/* Do not attempt EAS if schedutil is not being used. */
		policy = cpufreq_cpu_get(i);
		if (!policy)
			goto free;
		gov = policy->governor;
		cpufreq_cpu_put(policy);
		if (gov != &schedutil_gov) {
			if (rd->pd)
				pr_warn("rd %*pbl: Disabling EAS, schedutil is mandatory\n",
						cpumask_pr_args(cpu_map));
			goto free;
		}

		/* Create the new pd and add it to the local list. */
		tmp = pd_init(i);
		if (!tmp)
			goto free;
		tmp->next = pd;
		pd = tmp;

		/*
		 * Count performance domains and performance states for the
		 * complexity check.
		 */
		nr_pd++;
		nr_ps += em_pd_nr_perf_states(pd->em_pd);
	}

	/* Bail out if the Energy Model complexity is too high. */
	if (nr_pd * (nr_ps + nr_cpus) > EM_MAX_COMPLEXITY) {
		WARN(1, "rd %*pbl: Failed to start EAS, EM complexity is too high\n",
						cpumask_pr_args(cpu_map));
		goto free;
	}

	perf_domain_debug(cpu_map, pd);

	/* Attach the new list of performance domains to the root domain. */
	tmp = rd->pd;
	rcu_assign_pointer(rd->pd, pd);
	if (tmp)
		call_rcu(&tmp->rcu, destroy_perf_domain_rcu);

	return !!pd;

free:
	free_pd(pd);
	tmp = rd->pd;
	rcu_assign_pointer(rd->pd, NULL);
	if (tmp)
		call_rcu(&tmp->rcu, destroy_perf_domain_rcu);

	return false;

}
```

E2000Q 5.10内核，perf_domain_debug 打印信息
```bash
[    2.574534] root_domain 0-3: pd3:{ cpus=3 nr_pstate=4 }
[    2.574540] freq: 250000, power: 79, cost: 632
[    2.579072] freq: 500000, power: 197, cost: 788
[    2.583690] freq: 1000000, power: 409, cost: 818
[    2.588390] freq: 2000000, power: 839, cost: 839
[    2.593094]  pd2:{ cpus=2 nr_pstate=4 }
[    2.593096] freq: 250000, power: 79, cost: 632
[    2.601445] freq: 500000, power: 197, cost: 788
[    2.606054] freq: 1000000, power: 409, cost: 818
[    2.610749] freq: 2000000, power: 839, cost: 839
[    2.615445]  pd0:{ cpus=0-1 nr_pstate=4 }
[    2.615447] freq: 187500, power: 1, cost: 8
[    2.623709] freq: 375000, power: 9, cost: 36
[    2.628058] freq: 750000, power: 55, cost: 110
[    2.632579] freq: 1500000, power: 125, cost: 125
```

root_domain的overload和overutilized说明：
+ 对于一个 CPU 而言，其处于 overload 状态则说明其 rq 上有大于等于2个任务
+ 对于一个 CPU 而言，其处于 overutilized 状态说明该 cpu 的 utility 超过其 capacity（缺省预留20%的算力，另外，这里的 capacity 是用于cfs任务的算力）
+ 对于 root domain，overload 表示至少有一个 cpu 处于 overload 状态。overutilized 表示至少有一个 cpu 处于 overutilized 状态
+ overutilized 状态非常重要，它决定了调度器是否启用EAS，只有在系统没有 overutilized 的情况下EAS才会生效。overload和newidle balance的频次控制相关，当系统在overload的情况下，newidle balance才会启动进行均衡。

#### EAS能量计算方法

CPU在某个performance state(ps)下的计算能力：  
ps->cap = ps->freq * scale_cpu / cpu_max_freq	（1）

CPU在该频点performace state(ps)下的能量消耗：  
cpu_nrg = ps->power * cpu_util / ps->cap  （2）

结合(1) (2)可以得出CPU在该ps下的能量消耗
	cpu_nrg = ps->power * cpu_max_freq * cpu_util / ps->freq * scale_cpu (3)

其中 ps->power * cpu_max_freq / ps->freq 是一个固定数据存放在频点表的cost成员中

一个pd内的CPU，拥有相同的cost，所以一个pd内所有CPU的能量消耗可以表示为  
pd_nrg = ps->cost * sum(cpu_util) / scale_cpu

#### EAS的调度过程

在任务被重新唤醒或者fork新建时，会通过`select_task_rq_fair()`将任务进行balance，达到充分利用CPU的目的。在`select_task_rq_fair()`，若任务是被重新唤醒就会调用`find_energy_efficient_cpu()`进行选核执行
```c
/*
 * Predicts what cpu_util(@cpu) would return if @p was migrated (and enqueued)
 * to @dst_cpu.
 */
static unsigned long cpu_util_next(int cpu, struct task_struct *p, int dst_cpu)
{
	struct cfs_rq *cfs_rq = &cpu_rq(cpu)->cfs;
	// 计算cfs的util
	unsigned long util_est, util = READ_ONCE(cfs_rq->avg.util_avg);

	/*
	 * If @p migrates from @cpu to another, remove its contribution. Or,
	 * if @p migrates from another CPU to @cpu, add its contribution. In
	 * the other cases, @cpu is not impacted by the migration, so the
	 * util_avg should already be correct.
	 */
	// 在dst_cpu为-1的情况下
	// 若任务p运行在传入的CPU util = cfs_util - task_util(p)
	if (task_cpu(p) == cpu && dst_cpu != cpu)
		sub_positive(&util, task_util(p));
	else if (task_cpu(p) != cpu && dst_cpu == cpu)
		util += task_util(p);

	if (sched_feat(UTIL_EST)) {
		util_est = READ_ONCE(cfs_rq->avg.util_est.enqueued);

		/*
		 * During wake-up, the task isn't enqueued yet and doesn't
		 * appear in the cfs_rq->avg.util_est.enqueued of any rq,
		 * so just add it (if needed) to "simulate" what will be
		 * cpu_util() after the task has been enqueued.
		 */
		if (dst_cpu == cpu)
			util_est += _task_util_est(p);

		util = max(util, util_est);
	}

	return min(util, capacity_orig_of(cpu));
}


/*
 * compute_energy(): Estimates the energy that @pd would consume if @p was
 * migrated to @dst_cpu. compute_energy() predicts what will be the utilization
 * landscape of @pd's CPUs after the task migration, and uses the Energy Model
 * to compute what would be the energy if we decided to actually migrate that
 * task.
 */
// 计算任务迁移到dst_cpu后，整个pd，即此cluster的energy
static long
compute_energy(struct task_struct *p, int dst_cpu, struct perf_domain *pd)
{
	struct cpumask *pd_mask = perf_domain_span(pd);
	// 获取该CPU的算力，这里取得的是归一化到1024的算力
	unsigned long cpu_cap = arch_scale_cpu_capacity(cpumask_first(pd_mask));
	unsigned long max_util = 0, sum_util = 0;
	int cpu;

	// 对此pd中每个online cpu都执行计算
	for_each_cpu_and(cpu, pd_mask, cpu_online_mask) {
		// 这里计算util_cfs，当dst_cpu传入为-1时，这里是没将p的util计算进去的
		// 这个地方比较关键，p的util有没有算到cpu的util里面去是在这一步决定的
		unsigned long cpu_util, util_cfs = cpu_util_next(cpu, p, dst_cpu);
		struct task_struct *tsk = cpu == dst_cpu ? p : NULL;

		// 返回该CPU下cfs+irq+rt+dl使用掉的CPU算力总和
		// 这一步计算没将任务p的util加进去
		sum_util += schedutil_cpu_util(cpu, util_cfs, cpu_cap,
					       ENERGY_UTIL, NULL);
		// 这一步计算将任务p的util加进去了
		// 这里计算cpu_util的原因就是需要预测下一次该perf_domain的频率
		// 计算的cpu_util是略微放大了的，需要进一步分析这个放大的操作
		cpu_util = schedutil_cpu_util(cpu, util_cfs, cpu_cap,
					      FREQUENCY_UTIL, tsk);
		max_util = max(max_util, cpu_util);
	}

	// 计算该pd下所有CPU的功耗和
	return em_cpu_energy(pd->em_pd, max_util, sum_util);
}

// 计算该pd下所有CPU的功耗和
static inline unsigned long em_cpu_energy(struct em_perf_domain *pd,
				unsigned long max_util, unsigned long sum_util)
{
	unsigned long freq, scale_cpu;
	struct em_perf_state *ps;
	int i, cpu;

	/*
	 * In order to predict the performance state, map the utilization of
	 * the most utilized CPU of the performance domain to a requested
	 * frequency, like schedutil.
	 */
	cpu = cpumask_first(to_cpumask(pd->cpus));
	scale_cpu = arch_scale_cpu_capacity(cpu);
	ps = &pd->table[pd->nr_perf_states - 1];
	// 这里将perf_domai计算出来的最大的CPU利用率来推测CPU接下来需要调频的频率
	// 这里可以将freq称为推测频率
	// 计算公式 freq = 1.25 * max_f * max_util / scale_cpu
	// 这里是在最大频率1.25倍进行计算的
	freq = map_util_freq(max_util, ps->frequency, scale_cpu);

	/*
	 * Find the lowest performance state of the Energy Model above the
	 * requested frequency.
	 */
	// 将该频率映射到频率表上
	// 因为freq是在1.5倍最大频率计算的，算出来的推测频率可能大于最大频率
	// 假如大于最大频率的话，for循环执行后ps指向的就是最大频率了
	for (i = 0; i < pd->nr_perf_states; i++) {
		ps = &pd->table[i];
		if (ps->frequency >= freq)
			break;
	}

	/*
	 * The capacity of a CPU in the domain at the performance state (ps)
	 * can be computed as:
	 *
	 *             ps->freq * scale_cpu
	 *   ps->cap = --------------------                          (1)
	 *                 cpu_max_freq
	 *
	 * So, ignoring the costs of idle states (which are not available in
	 * the EM), the energy consumed by this CPU at that performance state
	 * is estimated as:
	 *
	 *             ps->power * cpu_util
	 *   cpu_nrg = --------------------                          (2)
	 *                   ps->cap
	 *
	 * since 'cpu_util / ps->cap' represents its percentage of busy time.
	 *
	 *   NOTE: Although the result of this computation actually is in
	 *         units of power, it can be manipulated as an energy value
	 *         over a scheduling period, since it is assumed to be
	 *         constant during that interval.
	 *
	 * By injecting (1) in (2), 'cpu_nrg' can be re-expressed as a product
	 * of two terms:
	 *
	 *             ps->power * cpu_max_freq   cpu_util
	 *   cpu_nrg = ------------------------ * ---------          (3)
	 *                    ps->freq            scale_cpu
	 *
	 * The first term is static, and is stored in the em_perf_state struct
	 * as 'ps->cost'.
	 *
	 * Since all CPUs of the domain have the same micro-architecture, they
	 * share the same 'ps->cost', and the same CPU capacity. Hence, the
	 * total energy of the domain (which is the simple sum of the energy of
	 * all of its CPUs) can be factorized as:
	 *
	 *            ps->cost * \Sum cpu_util
	 *   pd_nrg = ------------------------                       (4)
	 *                  scale_cpu
	 */
	// 根据推测频率在频点表上对应的cost来算energy
	// 不同频率的cost是常数，在初始化时就已经计算出来
	// 计算cost的时候需要将该perf_domain上的所有util拿出来进行计算
	// 一个perf_domain下的频率是一样的，所以这里计算能耗直接用了sum_util
	return ps->cost * sum_util / scale_cpu;
}

// 寻找工作能耗最低的CPU
static int find_energy_efficient_cpu(struct task_struct *p, int prev_cpu)
{
	unsigned long prev_delta = ULONG_MAX, best_delta = ULONG_MAX;
	struct root_domain *rd = cpu_rq(smp_processor_id())->rd;
	unsigned long cpu_cap, util, base_energy = 0;
	int cpu, best_energy_cpu = prev_cpu;
	struct sched_domain *sd;
	struct perf_domain *pd;

	rcu_read_lock();
	// 从rd取pd的指针
	pd = rcu_dereference(rd->pd);
	if (!pd || READ_ONCE(rd->overutilized))
		goto fail;

	/*
	 * Energy-aware wake-up happens on the lowest sched_domain starting
	 * from sd_asym_cpucapacity spanning over this_cpu and prev_cpu.
	 */
	sd = rcu_dereference(*this_cpu_ptr(&sd_asym_cpucapacity));
	while (sd && !cpumask_test_cpu(prev_cpu, sched_domain_span(sd)))
		sd = sd->parent;
	if (!sd)
		goto fail;

	sync_entity_load_avg(&p->se);
	// p的util为0，直接返回prev_cpu
	if (!task_util_est(p))
		goto unlock;

	// 遍历整个pd链表，计算p在不同pd下的能耗
	for (; pd; pd = pd->next) {
		unsigned long cur_delta, spare_cap, max_spare_cap = 0;
		unsigned long base_energy_pd;
		int max_spare_cap_cpu = -1;

		/* Compute the 'base' energy of the pd, without @p */
		// 计算不包括p的情况下此pd当前的energy
		// 这里传入dst_cpu为-1，计算的就是该perf_domain不包括任务P的util
		// 来进行频率推测，然后用来计算能耗
		base_energy_pd = compute_energy(p, -1, pd);
		// 不包括p的情况下系统的总energy
		base_energy += base_energy_pd;

		// 遍历整个pd中的CPU，计算p放在该CPU上的功耗
		for_each_cpu_and(cpu, perf_domain_span(pd), sched_domain_span(sd)) {
			if (!cpumask_test_cpu(cpu, p->cpus_ptr))
				continue;
			// 计算p放到此CPU后该CPU总共消耗的算力
			util = cpu_util_next(cpu, p, cpu);
			// 这里取的是归一化后的CPU算力
			cpu_cap = capacity_of(cpu);
			spare_cap = cpu_cap;
			// 计算p放到此CPU后剩余的算力
			lsub_positive(&spare_cap, util);

			/*
			 * Skip CPUs that cannot satisfy the capacity request.
			 * IOW, placing the task there would make the CPU
			 * overutilized. Take uclamp into account to see how
			 * much capacity we can get out of the CPU; this is
			 * aligned with schedutil_cpu_util().
			 */
			util = uclamp_rq_util_with(cpu_rq(cpu), util, p);
			// CPU需要保留20%左右的算力，不满足需求后进行下一个CPU的探测
			if (!fits_capacity(util, cpu_cap))
				continue;

			/* Always use prev_cpu as a candidate. */
			// 若对比的这个CPU就是任务之前运行的CPU
			if (cpu == prev_cpu) {
				// 计算p放在该cpu后整个pd的能量消耗
				// 这里传入的dst_cpu为之前p运行的CPU
				// 推测频率是在之前运行CPU进行推测，同时把P的util也计算到了
				prev_delta = compute_energy(p, prev_cpu, pd);
				// 计算p放在该CPU后整个pd增加的能量消耗
				prev_delta -= base_energy_pd;
				// 更新best_delta，取最优能耗
				best_delta = min(best_delta, prev_delta);
			}

			/*
			 * Find the CPU with the maximum spare capacity in
			 * the performance domain
			 */
			// 记录p放上去后剩余算力最大的CPU和最大的剩余算力
			if (spare_cap > max_spare_cap) {
				max_spare_cap = spare_cap;
				max_spare_cap_cpu = cpu;
			}
		}

		/* Evaluate the energy impact of using this CPU. */
		// 同一个簇上的CPU取最大余量算力的那个CPU与其他簇的CPU做能量消耗对比
		if (max_spare_cap_cpu >= 0 && max_spare_cap_cpu != prev_cpu) {
			// 计算p放在算力剩余最大的CPU后整个pd的能量消耗
			// 这里传进去的dst_cpu肯定不会和当前任务p运行的CPU相同
			// 所以在cpu_util_next()中会把p的util加到cpu util上
			cur_delta = compute_energy(p, max_spare_cap_cpu, pd);
			// 计算能量消耗增量
			cur_delta -= base_energy_pd;
			// 如果当前能量增量优于p放在prev_cpu运行的能量消耗，则取该cpu运行p
			if (cur_delta < best_delta) {
				best_delta = cur_delta;
				best_energy_cpu = max_spare_cap_cpu;
			}
		}
	}
unlock:
	rcu_read_unlock();

	/*
	 * Pick the best CPU if prev_cpu cannot be used, or if it saves at
	 * least 6% of the energy used by prev_cpu.
	 */
	// 若prev_cpu找不到，就直接返回最优能耗cpu
	if (prev_delta == ULONG_MAX)
		return best_energy_cpu;

	// 若最优能耗比放在prev_cpu上运行的能耗还要低6.25%以上，则取最优能耗cpu
	if ((prev_delta - best_delta) > ((prev_delta + base_energy) >> 4))
		return best_energy_cpu;

	// 否则不做改变，直接使用prev_cpu运行p
	return prev_cpu;

fail:
	rcu_read_unlock();

	return -1;
}
```

### EAS Mainline
https://git.gitlab.arm.com/linux-arm/linux-power.git