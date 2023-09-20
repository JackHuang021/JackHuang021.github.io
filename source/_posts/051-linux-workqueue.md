---
title: Linux workqueue实现
date: 2023-09-08 15:36:52
tags:
  - Linux
  - workqueue
categories: Linux
---

### 概述
工作队列（workqueue）是除软中断softirq和tasklet以外最常用的一种中断下半部执行机制，由内核统一管理。**工作队列把推迟执行的任务交给内核线程来执行，其运行在进程上下文，允许重新调度，睡眠，这个内核线程被称为worker thread**工作队列解决了软中断和tasklet执行时间过长导致系统实时性下降的问题，同时避免了驱动模块自身创建线程导致内核线程过多的问题。

<!-- more -->
工作队列的设计思想：
+ 并行，多个work不要相互阻塞
+ 节省资源，多个work尽量共享资源


### CMWQ
为了实现设计思想，工作队列的设计实现也更新了很多版本，最新的workqueue实现叫做CMWQ（Concurrency Managed Workqueue），CMWQ提出了worker pool的概念，系统中存在若干worker pool，这些worker pool不和特定的workqueue关联，而是所有的workqueue共享

用户可以创建workqueue，但不创建worker pool，并通过flag来约束挂入该workqueue上的work的处理方式，workqueue会根据其flag将work交付给系统中的某个worker pool处理，根据`struct workqueue_struct`中的flags成员来判断work的去向

CMWQ的几个基本概念
+ work：工作
+ workqueue：工作的集合，workqueue和work是一对多的关系
+ worker：工人，在代码中worker对应一个work_thread()内核线程
+ worker_pool：工人的集合，worker_pool和worker是一对多的关系
+ pwq（pool_workqueue）：中间人，负责建立起workqueue和worker_pool之间的关系，workqueue和pwq是一对多的关系，pwq和worker_pool是一对一的关系


CMWQ创建线程池的策略，系统中的线程池thread pool包括两种，一种是和特定CPU绑定的线程池percpu thread pool，另一种是unbound thread pool：
+ 针对percpu thread pool，每个CPU包含两个这样的线程池，一个是普通优先级的normal thread pool，一个是高优先级的high priority pool
+ unbound thread pool，它可以运行在任意的CPU上，这种线程池是动态创建的，根据`struct workqueue_attrs *unbound_attrs;`这个属性来创建unbound thread pool

CMWQ线程创建的策略：当thread pool中处于运行状态的worker thread等于0，并且有需要处理的work的时候，thread pool就会创建新的worker线程，当worker线程处于idle的时候，也不会立即销毁它，而是保持一段时间，如果这时候有创建新的worker需求的时候，那么直接wakeup idle的worker即可，一段时间过去仍然没有需要处理的work，那么该worker thread将会被销毁


### CMWQ相关结构体
worker thread pool使用`struct worker_pool`结构体用来描述
```c
// kernel/workqueue.c

/*
 * Structure fields follow one of the following exclusion rules.
 *
 * I: Modifiable by initialization/destruction paths and read-only for
 *    everyone else.
 *
 * P: Preemption protected.  Disabling preemption is enough and should
 *    only be modified and accessed from the local cpu.
 *
 * L: pool->lock protected.  Access with pool->lock held.
 *
 * X: During normal operation, modification requires pool->lock and should
 *    be done only from local cpu.  Either disabling preemption on local
 *    cpu or grabbing pool->lock is enough for read access.  If
 *    POOL_DISASSOCIATED is set, it's identical to L.
 *
 * A: wq_pool_attach_mutex protected.
 *
 * PL: wq_pool_mutex protected.
 *
 * PR: wq_pool_mutex protected for writes.  RCU protected for reads.
 *
 * PW: wq_pool_mutex and wq->mutex protected for writes.  Either for reads.
 *
 * PWR: wq_pool_mutex and wq->mutex protected for writes.  Either or
 *      RCU for reads.
 *
 * WQ: wq->mutex protected.
 *
 * WR: wq->mutex protected for writes.  RCU protected for reads.
 *
 * MD: wq_mayday_lock protected.
 */

struct worker_pool {
	raw_spinlock_t		lock;		/* the pool lock */
	int			cpu;		/* I: the associated cpu */
	int			node;		/* I: the associated node ID */
	int			id;		/* I: pool ID */
	unsigned int		flags;		/* X: flags */

	unsigned long		watchdog_ts;	/* L: watchdog timestamp */

	/*
	 * The counter is incremented in a process context on the associated CPU
	 * w/ preemption disabled, and decremented or reset in the same context
	 * but w/ pool->lock held. The readers grab pool->lock and are
	 * guaranteed to see if the counter reached zero.
	 */
	int			nr_running;

	struct list_head	worklist;	/* L: list of pending works */

	int			nr_workers;	/* L: total number of workers */
	int			nr_idle;	/* L: currently idle workers */

	struct list_head	idle_list;	/* L: list of idle workers */
	struct timer_list	idle_timer;	/* L: worker idle timeout */
	struct timer_list	mayday_timer;	/* L: SOS timer for workers */

	/* a workers is either on busy_hash or idle_list, or the manager */
	DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
						/* L: hash of busy workers */

	struct worker		*manager;	/* L: purely informational */
	struct list_head	workers;	/* A: attached workers */
	struct completion	*detach_completion; /* all workers detached */

	struct ida		worker_ida;	/* worker IDs for task name */

	struct workqueue_attrs	*attrs;		/* I: worker attributes */
	struct hlist_node	hash_node;	/* PL: unbound_pool_hash node */
	int			refcnt;		/* PL: refcnt for unbound pools */

	/*
	 * Destruction of pool is RCU protected to allow dereferences
	 * from get_work_pool().
	 */
	struct rcu_head		rcu;
};
```

工作队列的工作任务是用`struct work_struct`描述的，是工作队列处理工作任务的最小单位
```c
// 回调函数的定义
typedef void (*work_func_t)(struct work_struct *work);

struct work_struct {
	atomic_long_t data;
  // 将工作任务链接起来
	struct list_head entry;
  // 处理工作任务的回调函数
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```

pool_workqueue结构体，连接workqueue和worker pool的中介
```c
/*
 * The per-pool workqueue.  While queued, the lower WORK_STRUCT_FLAG_BITS
 * of work_struct->data are used for flags and the remaining high bits
 * point to the pwq; thus, pwqs need to be aligned at two's power of the
 * number of flag bits.
 */
struct pool_workqueue {
	struct worker_pool	*pool;		/* I: the associated pool */
	struct workqueue_struct *wq;		/* I: the owning workqueue */
	int			work_color;	/* L: current color */
	int			flush_color;	/* L: flushing color */
	int			refcnt;		/* L: reference count */
	int			nr_in_flight[WORK_NR_COLORS];
						/* L: nr of in_flight works */

	/*
	 * nr_active management and WORK_STRUCT_INACTIVE:
	 *
	 * When pwq->nr_active >= max_active, new work item is queued to
	 * pwq->inactive_works instead of pool->worklist and marked with
	 * WORK_STRUCT_INACTIVE.
	 *
	 * All work items marked with WORK_STRUCT_INACTIVE do not participate
	 * in pwq->nr_active and all work items in pwq->inactive_works are
	 * marked with WORK_STRUCT_INACTIVE.  But not all WORK_STRUCT_INACTIVE
	 * work items are in pwq->inactive_works.  Some of them are ready to
	 * run in pool->worklist or worker->scheduled.  Those work itmes are
	 * only struct wq_barrier which is used for flush_work() and should
	 * not participate in pwq->nr_active.  For non-barrier work item, it
	 * is marked with WORK_STRUCT_INACTIVE iff it is in pwq->inactive_works.
	 */
	int			nr_active;	/* L: nr of active works */
	int			max_active;	/* L: max active works */
	struct list_head	inactive_works;	/* L: inactive works */
	struct list_head	pwqs_node;	/* WR: node on wq->pwqs */
	struct list_head	mayday_node;	/* MD: node on wq->maydays */

	/*
	 * Release of unbound pwq is punted to system_wq.  See put_pwq()
	 * and pwq_unbound_release_workfn() for details.  pool_workqueue
	 * itself is also RCU protected so that the first pwq can be
	 * determined without grabbing wq->mutex.
	 */
	struct work_struct	unbound_release_work;
	struct rcu_head		rcu;
} __aligned(1 << WORK_STRUCT_FLAG_BITS);

```

workqueue使用workqueue_struct结构体来实现
```c
/*
 * The externally visible workqueue.  It relays the issued work items to
 * the appropriate worker_pool through its pool_workqueues.
 */
struct workqueue_struct {
	struct list_head	pwqs;		/* WR: all pwqs of this wq */
	struct list_head	list;		/* PR: list of all workqueues */

	struct mutex		mutex;		/* protects this wq */
	int			work_color;	/* WQ: current work color */
	int			flush_color;	/* WQ: current flush color */
	atomic_t		nr_pwqs_to_flush; /* flush in progress */
	struct wq_flusher	*first_flusher;	/* WQ: first flusher */
	struct list_head	flusher_queue;	/* WQ: flush waiters */
	struct list_head	flusher_overflow; /* WQ: flush overflow list */

	struct list_head	maydays;	/* MD: pwqs requesting rescue */
	struct worker		*rescuer;	/* MD: rescue worker */

	int			nr_drainers;	/* WQ: drain in progress */
	int			saved_max_active; /* WQ: saved pwq max_active */

	struct workqueue_attrs	*unbound_attrs;	/* PW: only for unbound wqs */
	struct pool_workqueue	*dfl_pwq;	/* PW: only for unbound wqs */

#ifdef CONFIG_SYSFS
	struct wq_device	*wq_dev;	/* I: for sysfs interface */
#endif
#ifdef CONFIG_LOCKDEP
	char			*lock_name;
	struct lock_class_key	key;
	struct lockdep_map	lockdep_map;
#endif
	char			name[WQ_NAME_LEN]; /* I: workqueue name */

	/*
	 * Destruction of workqueue_struct is RCU protected to allow walking
	 * the workqueues list without grabbing wq_pool_mutex.
	 * This is used to dump all workqueues from sysrq.
	 */
	struct rcu_head		rcu;

	/* hot fields used during command issue, aligned to cacheline */
	unsigned int		flags ____cacheline_aligned; /* WQ: WQ_* flags */
	struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
	struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */
};
```
+ list: 系统中所有的workqueue会挂入到一个全局链表`static LIST_HEAD(workqueues);`
+ cpu_pwqs：percpu workqueue指向percpu的pool_workqueue数据结构，用来维护workqueue和percpu thread pool之间的关系，每个CPU都有两个thread pool，normal和高优先级的线程池，cpu_pwqs指向哪一个pool_workqueue是和workqueue的flags相关的，如果标记有WQ_HIGHPRI，那么cpu_pwqs指向高优先级的线程池

### woker pool初始化
CMWQ对worker pool分成了两类：
+ percpu worker pool，给通用的workqueue使用，系统的规划是每个CPU创建两个worker pool，一个普通优先级(nice = 0)，一个高优先级(nice = HIGHPRI_NICE_LEVEL)，对应创建出来的worker线程nice值不一样
+ unbound woker pool，给WQ_UNBOUND类型的workqueue使用，unbound worker pool中的worer可以在多个CPU上调度，



workqueue子系统的初始化分为两个阶段，`workqueue_init_early()`在`start_kernel()`中调用，在这个阶段主要是做一些基本的初始化工作，例如对percpu worker thread pool的一些基本初始化
```c
// kernel/workqueue.c

// 这里给定义了一个长度为2的worker_pool数组，即percpu worker pool
// 普通优先级的worker pool为worker_pool[0]
// 高优先级的worker pool为worker_pool[1]
/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], cpu_worker_pools);

// 在workqueue_init_early()中会对percpu worker pool进行初始化工作
/**
 * workqueue_init_early - early init for workqueue subsystem
 *
 * This is the first half of two-staged workqueue subsystem initialization
 * and invoked as soon as the bare basics - memory allocation, cpumasks and
 * idr are up.  It sets up all the data structures and system workqueues
 * and allows early boot code to create workqueues and queue/cancel work
 * items.  Actual work item execution starts only after kthreads can be
 * created and scheduled right before early initcalls.
 */
void __init workqueue_init_early(void)
{
	int std_nice[NR_STD_WORKER_POOLS] = { 0, HIGHPRI_NICE_LEVEL };
	int i, cpu;

	BUILD_BUG_ON(__alignof__(struct pool_workqueue) < __alignof__(long long));

	BUG_ON(!alloc_cpumask_var(&wq_unbound_cpumask, GFP_KERNEL));
	cpumask_copy(wq_unbound_cpumask, housekeeping_cpumask(HK_TYPE_WQ));
	cpumask_and(wq_unbound_cpumask, wq_unbound_cpumask, housekeeping_cpumask(HK_TYPE_DOMAIN));

	pwq_cache = KMEM_CACHE(pool_workqueue, SLAB_PANIC);

	/* initialize CPU pools */
	for_each_possible_cpu(cpu) {
		struct worker_pool *pool;

		i = 0;
		for_each_cpu_worker_pool(pool, cpu) {
			// init_worker_pool()主要是对percpu worker_pool的成员进行初步初始化
			BUG_ON(init_worker_pool(pool));
			pool->cpu = cpu;
			cpumask_copy(pool->attrs->cpumask, cpumask_of(cpu));
			pool->attrs->nice = std_nice[i++];
			pool->node = cpu_to_node(cpu);

			/* alloc pool ID */
			mutex_lock(&wq_pool_mutex);
			BUG_ON(worker_pool_assign_id(pool));
			mutex_unlock(&wq_pool_mutex);
		}
	}

	/* create default unbound and ordered wq attrs */
	for (i = 0; i < NR_STD_WORKER_POOLS; i++) {
		struct workqueue_attrs *attrs;

		BUG_ON(!(attrs = alloc_workqueue_attrs()));
		// 默认普通优先级worker pool的nice值为0
		// 高优先级的worker pool的nice值为-20
		attrs->nice = std_nice[i];
		unbound_std_wq_attrs[i] = attrs;

		/*
		 * An ordered wq should have only one pwq as ordering is
		 * guaranteed by max_active which is enforced by pwqs.
		 * Turn off NUMA so that dfl_pwq is used for all nodes.
		 */
		BUG_ON(!(attrs = alloc_workqueue_attrs()));
		attrs->nice = std_nice[i];
		attrs->no_numa = true;
		ordered_wq_attrs[i] = attrs;
	}

	// 系统创建了一些默认的workqueue以供使用
	system_wq = alloc_workqueue("events", 0, 0);
	system_highpri_wq = alloc_workqueue("events_highpri", WQ_HIGHPRI, 0);
	system_long_wq = alloc_workqueue("events_long", 0, 0);
	system_unbound_wq = alloc_workqueue("events_unbound", WQ_UNBOUND,
					    WQ_UNBOUND_MAX_ACTIVE);
	system_freezable_wq = alloc_workqueue("events_freezable",
					      WQ_FREEZABLE, 0);
	system_power_efficient_wq = alloc_workqueue("events_power_efficient",
					      WQ_POWER_EFFICIENT, 0);
	system_freezable_power_efficient_wq = alloc_workqueue("events_freezable_power_efficient",
					      WQ_FREEZABLE | WQ_POWER_EFFICIENT,
					      0);
	BUG_ON(!system_wq || !system_highpri_wq || !system_long_wq ||
	       !system_unbound_wq || !system_freezable_wq ||
	       !system_power_efficient_wq ||
	       !system_freezable_power_efficient_wq);
}
```

workqueue第二阶段的初始化，经过这一阶段的初始化，workqueue就能正常使用了
```c
/**
 * workqueue_init - bring workqueue subsystem fully online
 *
 * This is the latter half of two-staged workqueue subsystem initialization
 * and invoked as soon as kthreads can be created and scheduled.
 * Workqueues have been created and work items queued on them, but there
 * are no kworkers executing the work items yet.  Populate the worker pools
 * with the initial workers and enable future kworker creations.
 */
void __init workqueue_init(void)
{
	struct workqueue_struct *wq;
	struct worker_pool *pool;
	int cpu, bkt;

	/*
	 * It'd be simpler to initialize NUMA in workqueue_init_early() but
	 * CPU to node mapping may not be available that early on some
	 * archs such as power and arm64.  As per-cpu pools created
	 * previously could be missing node hint and unbound pools NUMA
	 * affinity, fix them up.
	 *
	 * Also, while iterating workqueues, create rescuers if requested.
	 */
	wq_numa_init();

	mutex_lock(&wq_pool_mutex);

	// 对于非NUMA架构的CPU这个node固定为0
	for_each_possible_cpu(cpu) {
		for_each_cpu_worker_pool(pool, cpu) {
			pool->node = cpu_to_node(cpu);
		}
	}

	list_for_each_entry(wq, &workqueues, list) {
		wq_update_unbound_numa(wq, smp_processor_id(), true);
		WARN(init_rescuer(wq),
		     "workqueue: failed to create early rescuer for %s",
		     wq->name);
	}

	mutex_unlock(&wq_pool_mutex);

	/* create the initial workers */
	// 针对percpu worker thread pool创建一个worker
	for_each_online_cpu(cpu) {
		for_each_cpu_worker_pool(pool, cpu) {
			pool->flags &= ~POOL_DISASSOCIATED;
			BUG_ON(!create_worker(pool));
		}
	}
	// unbound woker pool也需要创建一个worker
	hash_for_each(unbound_pool_hash, bkt, pool, hash_node)
		BUG_ON(!create_worker(pool));

	wq_online = true;
	wq_watchdog_init();
}
```

接下来分析一下worker的创建过程
```c
/**
 * create_worker - create a new workqueue worker
 * @pool: pool the new worker will belong to
 *
 * Create and start a new worker which is attached to @pool.
 *
 * CONTEXT:
 * Might sleep.  Does GFP_KERNEL allocations.
 *
 * Return:
 * Pointer to the newly created worker.
 */
static struct worker *create_worker(struct worker_pool *pool)
{
	struct worker *worker;
	int id;
	char id_buf[16];

	/* ID is needed to determine kthread name */
	// 这里用到了ida的分配机制
	id = ida_alloc(&pool->worker_ida, GFP_KERNEL);
	if (id < 0)
		return NULL;

	// 为worker分配内存，进行一些成员的初始化
	// 这里分配内存使用到了kzalloc_node()实际也是为numa考虑的
	worker = alloc_worker(pool->node);
	if (!worker)
		goto fail;

	worker->id = id;

	// 线程名
	if (pool->cpu >= 0)
		snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, id,
			 pool->attrs->nice < 0  ? "H" : "");
	else
		snprintf(id_buf, sizeof(id_buf), "u%d:%d", pool->id, id);

	// 创建线程，线程函数worker_thread
	worker->task = kthread_create_on_node(worker_thread, worker, pool->node,
					      "kworker/%s", id_buf);
	if (IS_ERR(worker->task))
		goto fail;

	set_user_nice(worker->task, pool->attrs->nice);
	kthread_bind_mask(worker->task, pool->attrs->cpumask);

	/* successful, attach the worker to the pool */
	// 将worker挂到worker pool的workers链表下
	worker_attach_to_pool(worker, pool);

	/* start the newly created worker */
	raw_spin_lock_irq(&pool->lock);
	worker->pool->nr_workers++;
	worker_enter_idle(worker);
	wake_up_process(worker->task);
	raw_spin_unlock_irq(&pool->lock);

	return worker;

fail:
	ida_free(&pool->worker_ida, id);
	kfree(worker);
	return NULL;
}
```

关于unbind workqueue的功耗节省：当workqueue收到一个要处理的work，如果该workqueue是unbound类型的话，那么该work由unbound thread pool处理并调度执行的策略交给系统的调度器模块来完成，对于scheduler而言，它会考虑CPU的idle状态，从而尽可能让CPU保持在idle状态，从而节省能耗。因此，如果一个workqueue有WQ_UNBOUND这样的flag，则说明该workqueue上挂入的work处理是考虑到power saving的。如果workqueue没有WQ_UNBOUND flag，则说明该workqueue是percpu的，这时候，调度哪一个CPU运行worker thread来处理work已经不是scheduler可以控制的了，这样，也就间接影响了功耗。


### workqueue创建过程
workqueue的创建过程`alloc_workqueue()`
```c
// kernel/workqueue.c
struct workqueue_struct *alloc_workqueue(const char *fmt,
					 unsigned int flags,
					 int max_active, ...)
{
	size_t tbl_size = 0;
	va_list args;
	struct workqueue_struct *wq;
	struct pool_workqueue *pwq;

	/*
	 * Unbound && max_active == 1 used to imply ordered, which is no
	 * longer the case on NUMA machines due to per-node pools.  While
	 * alloc_ordered_workqueue() is the right way to create an ordered
	 * workqueue, keep the previous behavior to avoid subtle breakages
	 * on NUMA.
	 */
	if ((flags & WQ_UNBOUND) && max_active == 1)
		flags |= __WQ_ORDERED;

	/* see the comment above the definition of WQ_POWER_EFFICIENT */
	// 这里WQ_POWER_EFFICIENT标志和workqueue.power_efficient这个内核参数都可以让
	// workqueue编程unbind workqueue从而降低能耗
	if ((flags & WQ_POWER_EFFICIENT) && wq_power_efficient)
		flags |= WQ_UNBOUND;

	/* allocate wq and format name */
	// 这里涉及到在NUMA中unbound workqueue的work thread在不同node之间迁移的问题
	// 在NUMA中，CPU访问不同node中的内存，访问速度是相差很大的
	// 实际上unbound workqueue实际上是创建了per node的pool_workqueue
	if (flags & WQ_UNBOUND)
		tbl_size = nr_node_ids * sizeof(wq->numa_pwq_tbl[0]);

	wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL);
	if (!wq)
		return NULL;

	// unbound_attrs属性是给unbound workqueue来决定work的的分配的
	if (flags & WQ_UNBOUND) {
		wq->unbound_attrs = alloc_workqueue_attrs();
		if (!wq->unbound_attrs)
			goto err_free_wq;
	}

	// 这里是取max_active后面的参数用来给workqueue取名字
	va_start(args, max_active);
	vsnprintf(wq->name, sizeof(wq->name), fmt, args);
	va_end(args);

	// max_active限制最大可以创建的worker thread的数目
	// 对于percpu workqueue最大可创建的worker thread是WQ_MAX_ACTIVE（512）
	// 对于unbound workqueue最大可创建的worker thread是CPU核心数的4倍WQ_UNBOUND_MAX_ACTIVE
	max_active = max_active ?: WQ_DFL_ACTIVE;
	max_active = wq_clamp_max_active(max_active, flags, wq->name);

	/* init wq */
	// 初始化workqueue的其他成员
	wq->flags = flags;
	wq->saved_max_active = max_active;
	mutex_init(&wq->mutex);
	atomic_set(&wq->nr_pwqs_to_flush, 0);
	INIT_LIST_HEAD(&wq->pwqs);
	INIT_LIST_HEAD(&wq->flusher_queue);
	INIT_LIST_HEAD(&wq->flusher_overflow);
	INIT_LIST_HEAD(&wq->maydays);

	wq_init_lockdep(wq);
	INIT_LIST_HEAD(&wq->list);

	if (alloc_and_link_pwqs(wq) < 0)
		goto err_unreg_lockdep;

	if (wq_online && init_rescuer(wq) < 0)
		goto err_destroy;

	if ((wq->flags & WQ_SYSFS) && workqueue_sysfs_register(wq))
		goto err_destroy;

	/*
	 * wq_pool_mutex protects global freeze state and workqueues list.
	 * Grab it, adjust max_active and add the new @wq to workqueues
	 * list.
	 */
	mutex_lock(&wq_pool_mutex);

	mutex_lock(&wq->mutex);
	for_each_pwq(pwq, wq)
		pwq_adjust_max_active(pwq);
	mutex_unlock(&wq->mutex);

	list_add_tail_rcu(&wq->list, &workqueues);

	mutex_unlock(&wq_pool_mutex);

	return wq;

err_unreg_lockdep:
	wq_unregister_lockdep(wq);
	wq_free_lockdep(wq);
err_free_wq:
	free_workqueue_attrs(wq->unbound_attrs);
	kfree(wq);
	return NULL;
err_destroy:
	destroy_workqueue(wq);
	return NULL;
}
EXPORT_SYMBOL_GPL(alloc_workqueue);


// 分配pool workqueue的内存，建立workqueue和pool workqueue的关系
static int alloc_and_link_pwqs(struct workqueue_struct *wq)
{
	bool highpri = wq->flags & WQ_HIGHPRI;
	int cpu, ret;

	// percpu workqueue的处理
	if (!(wq->flags & WQ_UNBOUND)) {
		// 为percpu workqueue分配一个pool_workqueue(用来连接worker pool和workqueue)
		wq->cpu_pwqs = alloc_percpu(struct pool_workqueue);
		if (!wq->cpu_pwqs)
			return -ENOMEM;

		for_each_possible_cpu(cpu) {
			struct pool_workqueue *pwq =
				per_cpu_ptr(wq->cpu_pwqs, cpu);
			// 每个pool_workqueue都有一个对应的worker thread pool
			struct worker_pool *cpu_pools =
				per_cpu(cpu_worker_pools, cpu);
			// 初始化pool_workqueue
			// 最重要的是设置其对应的workqueue和woker_pool
			// 在这里pool_workqueue和woker pool，workqueue关联上了
			init_pwq(pwq, wq, &cpu_pools[highpri]);

			mutex_lock(&wq->mutex);
			// link_pwq主要是将pool_workqueue挂入它所属的workqueue链表中
			link_pwq(pwq);
			mutex_unlock(&wq->mutex);
		}
		return 0;
	}

	cpus_read_lock();
	if (wq->flags & __WQ_ORDERED) {
		ret = apply_workqueue_attrs(wq, ordered_wq_attrs[highpri]);
		/* there should only be single pwq for ordering guarantee */
		WARN(!ret && (wq->pwqs.next != &wq->dfl_pwq->pwqs_node ||
			      wq->pwqs.prev != &wq->dfl_pwq->pwqs_node),
		     "ordering guarantee broken for workqueue %s\n", wq->name);
	} else {
		ret = apply_workqueue_attrs(wq, unbound_std_wq_attrs[highpri]);
	}
	cpus_read_unlock();

	return ret;
}

// 每个pool_workqueue都有一个对应的worker thread pool
// 对于percpu workqueue这里静态定义了如下的cpu_worker_pools
/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], cpu_worker_pools);
```



> 变参函数宏
> + va_list：定义在编译器的头文件中
> + va_start(args, fmt)：根据参数fmt的地址，获取fmt后面的参数地址，并保存在args指针变量中
> + va_end(args)：释放args指针，将其值赋为NULL


