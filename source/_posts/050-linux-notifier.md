---
title: Linux notify机制
tags:
  - Linux
  - notifier
categories: Linux
abbrlink: 2f609589
date: 2023-09-07 15:38:46
---

在linux内核中，各个子系统之间有很强的相互关系，某些子系统可能对其他子系统产生的事件比较感兴趣，因此内核引进了notifier机制。内核实现notifier机制的代码位于`kernel/notifier.c`

<!-- more -->
内核使用`struct notifier_block`结构代表一个notifer
```c
// include/linux/notifier.h
typedef	int (*notifier_fn_t)(struct notifier_block *nb,
			unsigned long action, void *data);

struct notifier_block {
    // 代表时间发生之后调用的回调函数
	notifier_fn_t notifier_call;
    // 用来链接同一个类型的notifier
	struct notifier_block __rcu *next;
    // notifier chain的优先级，对应的数字越大优先级越高
	int priority;
};
```

内核同时也提供了四种不同类型的notifier chain
+ 原子通知链（atomic notifier chains）
```c
// include/linux/notifier.h
struct atomic_notifier_head {
	spinlock_t lock;
	struct notifier_block __rcu *head;
};
```

+ 可阻塞通知链（blocking notifier chains）
```c
// include/linux/notifier.h
struct blocking_notifier_head {
	struct rw_semaphore rwsem;
	struct notifier_block __rcu *head;
};
```

+ 原始通知链（raw notifier chains）
```c
// include/linux/notifier.h
struct raw_notifier_head {
	struct notifier_block __rcu *head;
};
```

+ SRCU通知链（SRCU notifier chains）
```c
// include/linux/notifier.h
struct srcu_notifier_head {
	struct mutex mutex;
	struct srcu_struct srcu;
	struct notifier_block __rcu *head;
};
```

通知链和notifier之间的组织关系如下
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230907193526.png)

内核提供了一套宏来初始化各个类型的通知链
```c
// include/linux/notifier.h
#define ATOMIC_INIT_NOTIFIER_HEAD(name) do {	\
		spin_lock_init(&(name)->lock);	\
		(name)->head = NULL;		\
	} while (0)
#define BLOCKING_INIT_NOTIFIER_HEAD(name) do {	\
		init_rwsem(&(name)->rwsem);	\
		(name)->head = NULL;		\
	} while (0)
#define RAW_INIT_NOTIFIER_HEAD(name) do {	\
		(name)->head = NULL;		\
	} while (0)

#define ATOMIC_NOTIFIER_INIT(name) {				\
		.lock = __SPIN_LOCK_UNLOCKED(name.lock),	\
		.head = NULL }
#define BLOCKING_NOTIFIER_INIT(name) {				\
		.rwsem = __RWSEM_INITIALIZER((name).rwsem),	\
		.head = NULL }
#define RAW_NOTIFIER_INIT(name)	{				\
		.head = NULL }

#define SRCU_NOTIFIER_INIT(name, pcpu)				\
	{							\
		.mutex = __MUTEX_INITIALIZER(name.mutex),	\
		.head = NULL,					\
		.srcu = __SRCU_STRUCT_INIT(name.srcu, pcpu),	\
	}

#define ATOMIC_NOTIFIER_HEAD(name)				\
	struct atomic_notifier_head name =			\
		ATOMIC_NOTIFIER_INIT(name)
#define BLOCKING_NOTIFIER_HEAD(name)				\
	struct blocking_notifier_head name =			\
		BLOCKING_NOTIFIER_INIT(name)
#define RAW_NOTIFIER_HEAD(name)					\
	struct raw_notifier_head name =				\
		RAW_NOTIFIER_INIT(name)
```

内核提供了最基本的注册通知链的函数，四个不同类型的通知链初始化通过调用这个基本的注册函数来进行注册
```c
// kernel/notifier.c
/*
 *	Notifier chain core routines.  The exported routines below
 *	are layered on top of these, with appropriate locking added.
 */

static int notifier_chain_register(struct notifier_block **nl,
				   struct notifier_block *n,
				   bool unique_priority)
{
    // 遍历nl通知链表，根据优先级大小将n插入到通知链表中
	while ((*nl) != NULL) {
		if (unlikely((*nl) == n)) {
			WARN(1, "notifier callback %ps already registered",
			     n->notifier_call);
			return -EEXIST;
		}
		if (n->priority > (*nl)->priority)
			break;
		if (n->priority == (*nl)->priority && unique_priority)
			return -EBUSY;
		nl = &((*nl)->next);
	}
    // 插入到
	n->next = *nl;
	rcu_assign_pointer(*nl, n);
	return 0;
}

static int notifier_chain_unregister(struct notifier_block **nl,
		struct notifier_block *n)
{
	while ((*nl) != NULL) {
		if ((*nl) == n) {
			rcu_assign_pointer(*nl, n->next);
			return 0;
		}
		nl = &((*nl)->next);
	}
	return -ENOENT;
}
```

接下来看一下如何发送一个事件到通知链上的notifier，内核提供了一个基本的实现`notifier_call_chain()`
```c
/**
 * notifier_call_chain - Informs the registered notifiers about an event.
 *	@nl:		Pointer to head of the blocking notifier chain
 *	@val:		Value passed unmodified to notifier function
 *	@v:		Pointer passed unmodified to notifier function
 *	@nr_to_call:	Number of notifier functions to be called. Don't care
 *			value of this parameter is -1.
 *	@nr_calls:	Records the number of notifications sent. Don't care
 *			value of this field is NULL.
 *	@returns:	notifier_call_chain returns the value returned by the
 *			last notifier function called.
 */
static int notifier_call_chain(struct notifier_block **nl,
			       unsigned long val, void *v,
			       int nr_to_call, int *nr_calls)
{
	int ret = NOTIFY_DONE;
	struct notifier_block *nb, *next_nb;

	nb = rcu_dereference_raw(*nl);

    // 遍历整个通知链，让通知链上的notifier全部调用一次回调函数
	while (nb && nr_to_call) {
		next_nb = rcu_dereference_raw(nb->next);

#ifdef CONFIG_DEBUG_NOTIFIERS
		if (unlikely(!func_ptr_is_kernel_text(nb->notifier_call))) {
			WARN(1, "Invalid notifier called!");
			nb = next_nb;
			continue;
		}
#endif
		ret = nb->notifier_call(nb, val, v);

		if (nr_calls)
			(*nr_calls)++;

		if (ret & NOTIFY_STOP_MASK)
			break;
		nb = next_nb;
		nr_to_call--;
	}
	return ret;
}
NOKPROBE_SYMBOL(notifier_call_chain);
```

原子通知链相关API实现
```c
// kernel/notifier.c

// 注册一个notifier block到通知链
/**
 *	atomic_notifier_chain_register - Add notifier to an atomic notifier chain
 *	@nh: Pointer to head of the atomic notifier chain
 *	@n: New entry in notifier chain
 *
 *	Adds a notifier to an atomic notifier chain.
 *
 *	Returns 0 on success, %-EEXIST on error.
 */
int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
		struct notifier_block *n)
{
	unsigned long flags;
	int ret;

	spin_lock_irqsave(&nh->lock, flags);
	ret = notifier_chain_register(&nh->head, n, false);
	spin_unlock_irqrestore(&nh->lock, flags);
	return ret;
}
EXPORT_SYMBOL_GPL(atomic_notifier_chain_register);

// 从通知链删除一个notifier
/**
 *	atomic_notifier_chain_unregister - Remove notifier from an atomic notifier chain
 *	@nh: Pointer to head of the atomic notifier chain
 *	@n: Entry to remove from notifier chain
 *
 *	Removes a notifier from an atomic notifier chain.
 *
 *	Returns zero on success or %-ENOENT on failure.
 */
int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,
		struct notifier_block *n)
{
	unsigned long flags;
	int ret;

	spin_lock_irqsave(&nh->lock, flags);
	ret = notifier_chain_unregister(&nh->head, n);
	spin_unlock_irqrestore(&nh->lock, flags);
	synchronize_rcu();
	return ret;
}
EXPORT_SYMBOL_GPL(atomic_notifier_chain_unregister);

/**
 *	atomic_notifier_call_chain - Call functions in an atomic notifier chain
 *	@nh: Pointer to head of the atomic notifier chain
 *	@val: Value passed unmodified to notifier function
 *	@v: Pointer passed unmodified to notifier function
 *
 *	Calls each function in a notifier chain in turn.  The functions
 *	run in an atomic context, so they must not block.
 *	This routine uses RCU to synchronize with changes to the chain.
 *
 *	If the return value of the notifier can be and'ed
 *	with %NOTIFY_STOP_MASK then atomic_notifier_call_chain()
 *	will return immediately, with the return value of
 *	the notifier function which halted execution.
 *	Otherwise the return value is the return value
 *	of the last notifier function called.
 */
int atomic_notifier_call_chain(struct atomic_notifier_head *nh,
			       unsigned long val, void *v)
{
	int ret;

	rcu_read_lock();
	ret = notifier_call_chain(&nh->head, val, v, -1, NULL);
	rcu_read_unlock();

	return ret;
}
EXPORT_SYMBOL_GPL(atomic_notifier_call_chain);
NOKPROBE_SYMBOL(atomic_notifier_call_chain);
```
  

