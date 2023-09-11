---
title: 051-linux-workqueue
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

为了实现设计思想，工作队列的设计实现也更新了很多版本，最新的workqueue实现叫做CMWQ（Concurrency Managed Workqueue）

CMWQ的几个基本概念
+ work：工作
+ workqueue：工作的集合，workqueue和work是一对多的关系
+ worker：工人，在代码中worker对应一个work_thread()内核线程
+ worker_pool：工人的集合，worker_pool和worker是一对多的关系
+ pwq（pool_workqueue）：中间人，负责建立起workqueue和worker_pool之间的关系，workqueue和pwq是一对多的关系，pwq和worker_pool是一对一的关系


最终的目的是要把work传递给worker去执行，每个执行work的线程叫做worker，一组worker的集合叫做worker_pool。



### 工作队列实现
#### worker_pool
系统的规划是每个CPU创建两个normal woker_pool
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

每个worker对应一个worker_therad()内核线程，一个worker_pool包含一个或者多个worker，worker_pool中worker的数量是根据worker_pool中的work负载来动态增减的


工作队列的工作任务是用`struct work_struct`描述的，是工作队列处理工作任务的最小单位
```c
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

workqueue_struct结构体
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
+ 