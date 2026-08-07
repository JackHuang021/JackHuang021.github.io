---
title: cpuidle framework
tags:
---


## 1. 软件架构

cpuidle 驱动框架位于 linux 内核 `driver/cpuidle` 文件夹下，包含 `cpuidle core`, `cpuidle governors`, `cpuidle_dirvers` 三个模块，再结合位于 kernel/sched 中的 idle 进程处理，共同完成 cpu 的 idle 状态管理。

![](https://raw.githubusercontent.com/JackHuang021/images/master/cpuidle.drawio.png)

### 1.1 kernel schedule

kernel schedule 模块位于 `kernel\sched\idle.c` 中，负责实现idle线程的通用入口（cpuidle entry）逻辑，包括 idle 模式的选择、idle 的进入等等。

### 1.2 cpuidle core

cpuidle core 负责实现 cpuidle framework 的整体框架，主要功能包括：

	+ 抽象出cpuidle device、cpuidle driver、cpuidle governor三个实体；
	+ 以函数调用的方式，向上层 sched 模块提供接口；
	+ 以 sysfs 的形式，向用户空间提供接口；
	+ 向下层的 cpuidle drivers 模块，提供统一的 driver 注册和管理接口；
	+ 向下层的 cpuilde governors 模块，提供统一的 governor 注册和管理接口。

### 1.3 cpuidle drivers

负责idle机制的实现，即：如何进入idle状态，什么条件下会退出，等等。不同的架构，会有不同的cpuidle driver。平台驱动的开发者，可以在cpuidle core提供的框架之下，开发自己的cpuidle driver。代码主要包括：cpuidle-xxx.c。

### 1.4 cpuidle governors

根据 cpuidle driver 提供的信息，governors 根据应用场景，决定cpu 进入哪种 idle 状态。代码位于 `drivers/cpuidle/governors/` 目录下。

## 2. 数据结构

### 2.1 `struct cpuidle_state`

linux 使用 `struct cpuidle_state` 结构抽象出不同级别的 idle state。

	```c
	// include/linux/cpuidle.h
	struct cpuidle_state {
		// cpuidle state 描述
		char		name[CPUIDLE_NAME_LEN];
		char		desc[CPUIDLE_DESC_LEN];
	
		// 从 idle 状态返回到运行状态的延迟，单位为 ns
		s64		exit_latency_ns;
		// 该 idle 状态期望的停留时间，进入和退出 idle 状态需要额外的能量消耗
		// 需要保持一定的停留时长来平衡消耗的能量
		s64		target_residency_ns;
		// idle 状态的标志
		unsigned int	flags;
		unsigned int	exit_latency; /* in US */
		// 在该 idle 状态下的功耗，单位为毫瓦
		int		power_usage; /* in mW */
		unsigned int	target_residency; /* in US */
		// 进入该 idle 状态的回调函数
		int (*enter)	(struct cpuidle_device *dev,
				struct cpuidle_driver *drv,
				int index);
		// CPU长时间不需要工作时（称作offline），可调用该回调函数。
		int (*enter_dead) (struct cpuidle_device *dev, int index);
	
		/*
		* CPUs execute ->enter_s2idle with the local tick or entire timekeeping
		* suspended, so it must not re-enable interrupts at any point (even
		* temporarily) or attempt to change states of clock event devices.
		*
		* This callback may point to the same function as ->enter if all of
		* the above requirements are met by it.
		*/
		int (*enter_s2idle)(struct cpuidle_device *dev,
					struct cpuidle_driver *drv,
					int index);
	};
	```

### 2.2 `struct cpuidle_device`

现实中，并没有 cpuidle device 这样一个真实的设备，因此 cpuidle device 是一个虚拟设备，我们可以把它类比为 cpu idle controller，负责实现cpuidle相关的逻辑。在多核CPU中，每个CPU core，都会对应一个cpuidle device。

	```c
	// include/linux/cpuidle.h
	struct cpuidle_device {
		// 标志该设备是否已注册到内核中
		unsigned int		registered:1;
		// 标志该设备是否已经使能
		unsigned int		enabled:1;
		unsigned int		poll_time_limit:1;
		// 该设备对应的 CPU 核心编号
		unsigned int		cpu;
		ktime_t			next_hrtimer;
	
		int			last_state_idx;
		// 上次停留在 idle 状态的时间
		u64			last_residency_ns;
		u64			poll_limit_ns;
		u64			forced_idle_latency_limit_ns;
		// 记录了该设备的每个 idle state 的统计信息，进入次数、停留时长等
		struct cpuidle_state_usage	states_usage[CPUIDLE_STATE_MAX];
			// 用于组织sysfs
		struct cpuidle_state_kobj *kobjs[CPUIDLE_STATE_MAX];
		struct cpuidle_driver_kobj *kobj_driver;
		struct cpuidle_device_kobj *kobj_dev;
		// 加入到 cpuidle_detected_devices 链表中，方便管理
		struct list_head 	device_list;
	
	#ifdef CONFIG_ARCH_NEEDS_CPU_IDLE_COUPLED
		cpumask_t		coupled_cpus;
		struct cpuidle_coupled	*coupled;
	#endif
	};


	struct cpuidle_state_usage {
		// 使能状态
		unsigned long long	disable;
		// idle状态进入次数
		unsigned long long	usage;
		// idle进入时间统计
		u64			time_ns;
		unsigned long long	above; /* Number of times it's been too deep */
		unsigned long long	below; /* Number of times it's been too shallow */
		unsigned long long	rejected; /* Number of times idle entry was rejected */
	#ifdef CONFIG_SUSPEND
		unsigned long long	s2idle_usage;
		unsigned long long	s2idle_time; /* in US */
	#endif
	};
	```

### 2.3 `struct cpuidle_driver`

cpuidle core 使用 `struct cpuidle_driver` 抽象 cpuidle 驱动。

	```c
	struct cpuidle_driver {
		const char		*name;
		struct module 		*owner;
	
		/* used by the cpuidle framework to setup the broadcast timer */
		// 用于指示在cpuidle driver注册和注销时，是否需要设置一个broadcast timer
		unsigned int            bctimer:1;
		/* states array must be ordered in decreasing power consumption */
		// idle state 状态信息
		struct cpuidle_state	states[CPUIDLE_STATE_MAX];
		// 该 driver 支持的 idle state 个数
		int			state_count;
		int			safe_state_index;
	
		/* the driver handles the cpus in cpumask */
		// 该驱动对应的CPU掩码
		struct cpumask		*cpumask;
	
		/* preferred governor to switch at register time */
		const char		*governor;
	};
	```

### 2.4 `struct cpuidle_governor`

cpuidle core 使用 `struct cpuidle_governor` 结构抽象 cpuidle governor。

	```c
	struct cpuidle_governor {
		// governor 的名称
		char			name[CPUIDLE_NAME_LEN];
		// 链接到 cpuidle_governors 全局链表中
		struct list_head 	governor_list;
		// governor 的级别，kernel 会选择 rating 值最大的 governor
		unsigned int		rating;
	
		// governor 使能、禁用回调函数
		int  (*enable)		(struct cpuidle_driver *drv,
						struct cpuidle_device *dev);
		void (*disable)		(struct cpuidle_driver *drv,
						struct cpuidle_device *dev);
		// 选择 idle state 的回调函数
		int  (*select)		(struct cpuidle_driver *drv,
						struct cpuidle_device *dev,
						bool *stop_tick);
		// 通过该回调函数，告知 governor，系统上一次所处的 idle state 是哪个，idle 状态结束后调用
		void (*reflect)		(struct cpuidle_device *dev, int index);
	};
	```

## 3. cpuidle 相关流程

### 3.1 cpuidle 框架的初始化

cpuidle framework的初始化，由 `cpuidle_init()` 实现，主要功能是：

+ 初始化 per-CPU idle 设备（cpuidle device）。

+ 创建 cpuidle sysfs。

+ 设置内核全局的 cpuidle 数据结构，准备 CPU idle 框架工作。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20251118160746.png)

### 3.2 cpuidle 调度流程梳理

在Linux系统启动的时候，会在每个 cpu 上创建对应的 idle 进程，`start_kernel()` 函数初始化内核需要的所有数据结构，并创建一个名为 init 的进程（pid=1），当 init 进程创建完后，cpu处于 `do_idle()` 无限循环中，当没有其他进程处于 TASK_RUNNING 状态时候，调度器才会执行 cpu idle 流程，让 cpu 进入 idle 状态。

	```c
	// kernel/sched/idle.c
	static void do_idle(void)
	{
		int cpu = smp_processor_id();
	
		/*
		* Check if we need to update blocked load
		*/
		nohz_run_idle_balance(cpu);
	
		/*
		* If the arch has a polling bit, we maintain an invariant:
		*
		* Our polling bit is clear if we're not scheduled (i.e. if rq->curr !=
		* rq->idle). This means that, if rq->idle has the polling bit set,
		* then setting need_resched is guaranteed to cause the CPU to
		* reschedule.
		*/
	
		__current_set_polling();
		tick_nohz_idle_enter();
		// need_resched() 判断当前 CPU 上正在运行的任务是否需要被重新调度。
		while (!need_resched()) {
			rmb();
	
			local_irq_disable();
	
			if (cpu_is_offline(cpu)) {
				tick_nohz_idle_stop_tick();
				cpuhp_report_idle_dead();
				arch_cpu_idle_dead();
			}
	
			arch_cpu_idle_enter();
			rcu_nocb_flush_deferred_wakeup();
	
			/*
			* In poll mode we reenable interrupts and spin. Also if we
			* detected in the wakeup from idle path that the tick
			* broadcast device expired for us, we don't want to go deep
			* idle as we know that the IPI is going to arrive right away.
			*/
			if (cpu_idle_force_poll || tick_check_broadcast_expired()) {
				tick_nohz_idle_restart_tick();
				cpu_idle_poll();
			} else {
				// cpu进入idle状态
				cpuidle_idle_call();
			}
			arch_cpu_idle_exit();
		}
	
		/*
		* Since we fell out of the loop above, we know TIF_NEED_RESCHED must
		* be set, propagate it into PREEMPT_NEED_RESCHED.
		*
		* This is required because for polling idle loops we will not have had
		* an IPI to fold the state for us.
		*/
		preempt_set_need_resched();
		tick_nohz_idle_exit();
		__current_clr_polling();
	
		/*
		* We promise to call sched_ttwu_pending() and reschedule if
		* need_resched() is set while polling is set. That means that clearing
		* polling needs to be visible before doing these things.
		*/
		smp_mb__after_atomic();
	
		/*
		* RCU relies on this call to be done outside of an RCU read-side
		* critical section.
		*/
		flush_smp_call_function_queue();
		schedule_idle();
	
		if (unlikely(klp_patch_pending(current)))
			klp_update_patch_state(current);
	}
	```

进入 idle 状态的调用路径为`cpuidle_idle_call() -> cpuidle_select() -> call_cpuildle() -> cpuidle_enter() -> target_state->enter()`，cpuidle 的调度是从调度层 → cpuidle 核心框架 → governor → 驱动 → idle state 对应的 enter() 接口，进入对应的idle状态。

## 4. acpi_ilde 驱动

### 4.1 LPI 介绍

从 ACPI 6.0 开始，引入新的 idle 模型 LPI（Low Power Idle），支持面向 ARM64 服务器、超低功耗 SoC，支持 vendor-specific 低功耗层级，允许更加复杂的电源域结构。
	![](https://raw.githubusercontent.com/JackHuang021/images/master/20251114171901.png)

ACPI 表中的每个 LPI package 的描述如下：
	![](https://raw.githubusercontent.com/JackHuang021/images/master/20251114171956.png)

	+ Min Residency: 该LPI状态要求CPU至少停留的时间，单位 us；
	+ Worst case wakeup latency： 从此状态下唤醒所需的最长时延，单位 us；
	+ Flags： 该值为 1 表示使能该状态
	+ Entry Method： 触发进入该低功耗状态的方法

D3000M 笔记本中的其中一个 LPI package 的描述如下：
	![](https://raw.githubusercontent.com/JackHuang021/images/master/20251114172124.png)

	该 LPI package 即表示：WFI 低功耗状态，最小停留时间为 500us，最大退出时延为 500us。

### 4.2 acpi_idle 加载过程

其过程主要是通过 acpi 表中的 LPI 描述来解析 CPU 的 idle 状态表，将数据记录到`cpuidle_driver->states`中，然后注册 cpuidle_driver 和 cpuidle_device。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20251119162133.png)

## 5. CPUidle Governor

关闭一些核可以节省功耗，但关闭之后对时延(性能)必会造成一定的影响，如果在关闭之后很短的时间内就被唤醒，那么就会造成功耗/性能双方都不讨好，在进入退出 idle 的过程中也是会有功耗的损失的，如果在 idle 状态下面节省的功耗还无法弥补进入退出该 idle 的功耗，那么反而会得不偿失。

根据性能/功耗的的这种矛盾，厂家会制定多个层级的 idle 状态，在每个层级下面的功耗、进入退出 idle 的功耗损失、以及进入退出的延迟都会是不同的数值，而 cpuidle framework 会根据不同的场景来进行仲裁选择使用何种的 idle 状态。

cpuidle governor的主要职责是决策一个最佳的idle state，主要的考量是基于切换的功耗代价和系统的延迟容忍度。

在当前的内核中，有两种主流的 governor 策略：ladder 和 menu，目前我们内核中默认配置的是 menu governor，适用于 tickless 系统。

**Ladder**：从字面上理解是阶梯式的策略，即要到更高的层级必须从低层级逐级进入，在 ladder 策略中，ladder governor 会首先进入最浅的 idle state，然后如果待的时间足够长，则会进入到更深一级的 idle state，以此类推，直到到达最深的 idle state，被唤醒时，会尽可能快地重新启动CPU；等到下次空闲，则又会从 idle state1 开始进入。在这种策略中，系统可能长时间都不进入最深的 idle state 中，造成一些功耗损失。

**Menu**：从字面上理解是菜单式的策略，即只要具备进入更深层次 idle state 的条件，系统就可以选择进入到该 idle state 中，不需要从浅到深逐层递进。menu会考虑 tickless 系统中的 sleep length（通过 tick_nohz_get_sleep_length() 获取），来选择最适合的 idle state。

### 5.1 menu governor

menu governor 的目标是：在保证唤醒延迟（exit latency）可接受的前提下，尽可能把 CPU 选进最深的 idle state。它以“预测下一次唤醒间隔”为核心，结合每个 state 的 target_residency 和 exit_latency 做选择。

初始化的时候会调用 cpuidle_register_governor() 来注册 menu_governor，提供了enable()、select()、reflect() 三个接口

	```c
	// drivers/cpuidle/governors/menu.c
	static struct cpuidle_governor menu_governor = {
		.name =		"menu",
		.rating =	20,
		.enable =	menu_enable_device,
		.select =	menu_select,
		.reflect =	menu_reflect,
	};
	
	struct menu_device {
		int             needs_update;
		int             tick_wakeup;
	
		u64		next_timer_ns;
		// 当前使用的校正因子
		unsigned int	bucket;
		// 保存的校正因子
		unsigned int	correction_factor[BUCKETS];
		// 保存之前的间隔时间，后续计算预测唤醒间隔时间时有用
		unsigned int	intervals[INTERVALS];
		int		interval_ptr;
	};
	```

#### 5.1.1 menu_select()

menu_select() 将计算出的 predicted_us 与所有 idle 状态的停留时间进行比较，选择特定idle 状态的条件是相应的停留时间应小于 predicted_us。另外，将状态的 exit_latency 与系统的交互性要求进行比较。基于两个等待时间因素，选择适当的 idle 状态。

	```c
	// drivers/cpuidle/governors/menu.c
	/**
	* menu_select - selects the next idle state to enter
	* @drv: cpuidle driver containing state data
	* @dev: the CPU
	* @stop_tick: indication on whether or not to stop the tick
	*/
	static int menu_select(struct cpuidle_driver *drv, struct cpuidle_device *dev,
				bool *stop_tick)
	{
		struct menu_device *data = this_cpu_ptr(&menu_devices);
		// 获取当前CPU的延迟限制时间，该时间来自 PM QoS 框架
		s64 latency_req = cpuidle_governor_latency_req(dev->cpu);
		// 预估的下次唤醒时间
		u64 predicted_ns;
		u64 interactivity_req;
		unsigned int nr_iowaiters;
		ktime_t delta, delta_tick;
		int i, idx;
	
		if (data->needs_update) {
			menu_update(drv, dev);
			data->needs_update = 0;
		}
		// 获取当前 CPU 上 IO wait 任务的个数
		nr_iowaiters = nr_iowait_cpu(dev->cpu);
	
		/* Find the shortest expected idle interval. */
		// 将之前的8个idle间隔时间的平均值作为predicted_us
		predicted_ns = get_typical_interval(data) * NSEC_PER_USEC;
		
		if (predicted_ns > RESIDENCY_THRESHOLD_NS) {
			unsigned int timer_us;
	
			/* Determine the time till the closest timer. */
			// 获取下次 timer 中断到来的时间，这个时间也是预估的
			delta = tick_nohz_get_sleep_length(&delta_tick);
			if (unlikely(delta < 0)) {
				delta = 0;
				delta_tick = 0;
			}
	
			data->next_timer_ns = delta;
			// 根据io wait任务个数和下一次中断时间来选择校正因子
			data->bucket = which_bucket(data->next_timer_ns, nr_iowaiters);
	
			/* Round up the result for half microseconds. */
			timer_us = div_u64((RESOLUTION * DECAY * NSEC_PER_USEC) / 2 +
						data->next_timer_ns *
							data->correction_factor[data->bucket],
					RESOLUTION * DECAY * NSEC_PER_USEC);
			/* Use the lowest expected idle interval to pick the idle state. */
			predicted_ns = min((u64)timer_us * NSEC_PER_USEC, predicted_ns);
		} else {
			/*
			* Because the next timer event is not going to be determined
			* in this case, assume that without the tick the closest timer
			* will be in distant future and that the closest tick will occur
			* after 1/2 of the tick period.
			*/
			data->next_timer_ns = KTIME_MAX;
			delta_tick = TICK_NSEC / 2;
			data->bucket = which_bucket(KTIME_MAX, nr_iowaiters);
		}
	
		if (unlikely(drv->state_count <= 1 || latency_req == 0) ||
			((data->next_timer_ns < drv->states[1].target_residency_ns ||
			latency_req < drv->states[1].exit_latency_ns) &&
			!dev->states_usage[0].disable)) {
			/*
			* In this case state[0] will be used no matter what, so return
			* it right away and keep the tick running if state[0] is a
			* polling one.
			*/
			*stop_tick = !(drv->states[0].flags & CPUIDLE_FLAG_POLLING);
			return 0;
		}
	
		if (tick_nohz_tick_stopped()) {
			/*
			* If the tick is already stopped, the cost of possible short
			* idle duration misprediction is much higher, because the CPU
			* may be stuck in a shallow idle state for a long time as a
			* result of it.  In that case say we might mispredict and use
			* the known time till the closest timer event for the idle
			* state selection.
			*/
			if (predicted_ns < TICK_NSEC)
				predicted_ns = data->next_timer_ns;
		} else {
			/*
			* Use the performance multiplier and the user-configurable
			* latency_req to determine the maximum exit latency.
			*/
			// 根据io wait的任务个数来计算延迟时间，计算公式为predicted_us / (1 +10 * iowaiters)
			interactivity_req = div64_u64(predicted_ns,
							performance_multiplier(nr_iowaiters));
			if (latency_req > interactivity_req)
				latency_req = interactivity_req;
		}
	
		/*
		* Find the idle state with the lowest power while satisfying
		* our constraints.
		*/
		// 根据预测下次唤醒时间和最大容忍延迟时间来挑选需要进入的 idle 状态
		idx = -1;
		for (i = 0; i < drv->state_count; i++) {
			struct cpuidle_state *s = &drv->states[i];
	
			if (dev->states_usage[i].disable)
				continue;
	
			if (idx == -1)
				idx = i; /* first enabled state */
	
			if (s->target_residency_ns > predicted_ns) {
				/*
				* Use a physical idle state, not busy polling, unless
				* a timer is going to trigger soon enough.
				*/
				if ((drv->states[idx].flags & CPUIDLE_FLAG_POLLING) &&
					s->exit_latency_ns <= latency_req &&
					s->target_residency_ns <= data->next_timer_ns) {
					predicted_ns = s->target_residency_ns;
					idx = i;
					break;
				}
				if (predicted_ns < TICK_NSEC)
					break;
	
				if (!tick_nohz_tick_stopped()) {
					/*
					* If the state selected so far is shallow,
					* waking up early won't hurt, so retain the
					* tick in that case and let the governor run
					* again in the next iteration of the loop.
					*/
					predicted_ns = drv->states[idx].target_residency_ns;
					break;
				}
	
				/*
				* If the state selected so far is shallow and this
				* state's target residency matches the time till the
				* closest timer event, select this one to avoid getting
				* stuck in the shallow one for too long.
				*/
				if (drv->states[idx].target_residency_ns < TICK_NSEC &&
					s->target_residency_ns <= delta_tick)
					idx = i;
	
				return idx;
			}
			if (s->exit_latency_ns > latency_req)
				break;
	
			idx = i;
		}
	
		if (idx == -1)
			idx = 0; /* No states enabled. Must use 0. */
	
		/*
		* Don't stop the tick if the selected state is a polling one or if the
		* expected idle duration is shorter than the tick period length.
		*/
		if (((drv->states[idx].flags & CPUIDLE_FLAG_POLLING) ||
			predicted_ns < TICK_NSEC) && !tick_nohz_tick_stopped()) {
			*stop_tick = false;
	
			if (idx > 0 && drv->states[idx].target_residency_ns > delta_tick) {
				/*
				* The tick is not going to be stopped and the target
				* residency of the state to be returned is not within
				* the time until the next timer event including the
				* tick, so try to correct that.
				*/
				for (i = idx - 1; i >= 0; i--) {
					if (dev->states_usage[i].disable)
						continue;
	
					idx = i;
					if (drv->states[i].target_residency_ns <= delta_tick)
						break;
				}
			}
		}
	
		return idx;
	}
	```

#### 5.1.2 menu_reflect()

上一个 idle 状态退出后，会调用 reflect() 接口来更新 menu governor 的参数，包括上次的idle 持续时间（保存到 intervals 数组） 和 校正因子。

	```c
	/**
	* menu_reflect - records that data structures need update
	* @dev: the CPU
	* @index: the index of actual entered state
	*
	* NOTE: it's important to be fast here because this operation will add to
	*       the overall exit latency.
	*/
	static void menu_reflect(struct cpuidle_device *dev, int index)
	{
		struct menu_device *data = this_cpu_ptr(&menu_devices);
	
		dev->last_state_idx = index;
		data->needs_update = 1;
		// idle过程是否被tick唤醒
		data->tick_wakeup = tick_nohz_idle_got_tick();
	}


	/**
	 * menu_update - attempts to guess what happened after entry
	* @drv: cpuidle driver containing state data
	* @dev: the CPU
	*/
	// menu 参数更新
	static void menu_update(struct cpuidle_driver *drv, struct cpuidle_device *dev)
	{
		struct menu_device *data = this_cpu_ptr(&menu_devices);
		int last_idx = dev->last_state_idx;
		struct cpuidle_state *target = &drv->states[last_idx];
		u64 measured_ns;
		unsigned int new_factor;
	
		/*
		* Try to figure out how much time passed between entry to low
		* power state and occurrence of the wakeup event.
		*
		* If the entered idle state didn't support residency measurements,
		* we use them anyway if they are short, and if long,
		* truncate to the whole expected time.
		*
		* Any measured amount of time will include the exit latency.
		* Since we are interested in when the wakeup begun, not when it
		* was completed, we must subtract the exit latency. However, if
		* the measured amount of time is less than the exit latency,
		* assume the state was never reached and the exit latency is 0.
		*/
	
		if (data->tick_wakeup && data->next_timer_ns > TICK_NSEC) {
			/*
			* The nohz code said that there wouldn't be any events within
			* the tick boundary (if the tick was stopped), but the idle
			* duration predictor had a differing opinion.  Since the CPU
			* was woken up by a tick (that wasn't stopped after all), the
			* predictor was not quite right, so assume that the CPU could
			* have been idle long (but not forever) to help the idle
			* duration predictor do a better job next time.
			*/
			measured_ns = 9 * MAX_INTERESTING / 10;
		} else if ((drv->states[last_idx].flags & CPUIDLE_FLAG_POLLING) &&
			dev->poll_time_limit) {
			/*
			* The CPU exited the "polling" state due to a time limit, so
			* the idle duration prediction leading to the selection of that
			* state was inaccurate.  If a better prediction had been made,
			* the CPU might have been woken up from idle by the next timer.
			* Assume that to be the case.
			*/
			measured_ns = data->next_timer_ns;
		} else {
			/* measured value */
			// idle状态退出后会计算该idle状态持续的时间，代码位于 cpuidle_enter_state()中
			measured_ns = dev->last_residency_ns;
	
			/* Deduct exit latency */
			if (measured_ns > 2 * target->exit_latency_ns)
				measured_ns -= target->exit_latency_ns;
			else
				measured_ns /= 2;
		}
	
		/* Make sure our coefficients do not exceed unity */
		if (measured_ns > data->next_timer_ns)
			measured_ns = data->next_timer_ns;
	
		/* Update our correction ratio */
		new_factor = data->correction_factor[data->bucket];
		new_factor -= new_factor / DECAY;
	
		if (data->next_timer_ns > 0 && measured_ns < MAX_INTERESTING)
			// 更新校正因子的数值，计算公式为 最近8次 实际的间隔时间 / 估计间隔时间 的平均值
			new_factor += div64_u64(RESOLUTION * measured_ns,
						data->next_timer_ns);
		else
			/*
			* we were idle so long that we count it as a perfect
			* prediction
			*/
			new_factor += RESOLUTION;
	
		/*
		* We don't want 0 as factor; we always want at least
		* a tiny bit of estimated time. Fortunately, due to rounding,
		* new_factor will stay nonzero regardless of measured_us values
		* and the compiler can eliminate this test as long as DECAY > 1.
		*/
		if (DECAY == 1 && unlikely(new_factor == 0))
			new_factor = 1;
		// 数据更新到menu_device中
		data->correction_factor[data->bucket] = new_factor;
	
		/* update the repeating-pattern data */
		// 将上次的间隔时间记录到intervals数组中
		data->intervals[data->interval_ptr++] = ktime_to_us(measured_ns);
		if (data->interval_ptr >= INTERVALS)
			data->interval_ptr = 0;
	}
	```

## 6. cpuidle 调试

1. 查看当前系统 cpuidle 的触发状态，usage 表示该 idle state 被触发的次数，time 表示累积停留时间。
	```bash
	cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
	cat /sys/devices/system/cpu/cpu0/cpuidle/state*/disable
	cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage
	cat /sys/devices/system/cpu/cpu0/cpuidle/state*/time
	```

2. 抓取内核 trace
	```bash
	echo 1 > /sys/kernel/debug/tracing/events/power/cpu_idle/enable
	cat /sys/kernel/debug/tracing/trace
	```

3. 观察调用 idle state 的 enter() 接口后，CPU是否执行了对应进入 idle 状态的指令

## 7. D3000M cpuidle

D3000M uefi 固件的 cpuidle 驱动为 acpi_idle，一共包含 4 个 idle 状态,分别为：

	+ WFI：让 CPU 暂停执行，直到出现中断或事件，以降低功耗。
	+ CoreRet：CPU 核心逻辑电路保留状态，但大部分电源被切掉
	+ CorePwrDn：核心电源关断
	+ CorePwrDn+CluPwrDn：核心电源关断并且 cluster 电源也关断

D3000M cpuidle states 表格：

| name | desc | latency | residency |
| :-: | :-: | :-: | :-: |
| LPI-0 | WFI | 500 | 500 |
| LPI-1 | CoreRet | 2500 | 2500 |
| LPI-2 | CorePwrDn | 5000 | 5000 |
| LPI-3 | CorePwrDn+CluPwrDn | 35000 | 30000 |

### 7.1 D3000M 笔记本 cpuidle 运行情况

在笔记本处于待机状态下，抓了一段 cpuidle 的 trace，可以看到除了当前正在跑 trace_cmd 进程的 CPU3，其它几个CPU基本都跑到了 LPI1 ~ LPI3 的 idle 状态。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20251119163405.png)
