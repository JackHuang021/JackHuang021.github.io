---
title: linux I/O 调度器
tags:
---

## 1. linux I/O调度器介绍
Linux操作系统需要I/O调度器（I/O Scheduler）来优化磁盘访问性能，平衡系统资源的使用，确保不同应用程序的I/O需求得到合理的满足。HDD磁盘有较高的寻道时间，即磁盘头移动到特定为止读取或写入数据所需要的时间。I/O调度器通过重新排序I/O请求来减少磁盘寻道的频率和距离，从而提高磁盘的吞吐量和性能。这和现实生活中的电梯模型比较类似，所以IO调度器也被称为电梯(elevator)，相应的算法被叫做电梯算法。

## 2. 块设备基础知识
### 2.1 块设备结构
+ 段： 由若干个相邻的块组成，是linux内存管理机制中一个内存页或者内存页的一部分，段在内核中由`struct bio_vec`来描述
+ 块： 块是虚拟文件系统传输数据的基本单位，通常由1个或多个扇区组成
+ 扇区： 扇区是硬件传输数据的基本单位，硬件一次传输一个扇区的数据，通常为512字节

### 2.2 块设备操作流程
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240903170933.png)

+ 虚拟文件系统层：VFS是一个抽象层，向上提供统一的文件访问接口，向下则兼容了各种不同的文件系统
+ 通用块层： 提供一个统一的接口让文件系统实现者使用，而不用关心不同设备驱动程序的差异，这样实现出来的文件系统就能用于任何的块设备
+ I/O调度层： 接收通用块层发出的I/O请求，对请求进行合并、排序，回调驱动层提供的请求处理函数

**块设备读写流程说明**
1. 文件系统向通用块层发起块读写请求
2. 通用块层(Generic Block Layer)将读写请求封装成`bio`结构体，下发到I/O调度层；I/O调度层将`bio`封装成`request`结构体，并将`request`添加到对应块设备`gendisk`的请求队列`request_queue`中，并做一些预处理（合并、排序）
+ I/O调度层对`request_queue`中的`request`进行预处理后，调用请求队列的回调函数对队列中的`request`进行处理
+ 请求队列的回调函数由块设备驱动层提供，块设备驱动层会从请求队列提取`request`并进行实质性的处理

### 2.3 重要的结构体
`struct bio`，用来描述单一的I/O请求，记录了一次I/O操作所必需的相关信息，
```c
// include/linux/blk_types.h
/*
 * main unit of I/O for the block layer and lower layers (ie drivers and
 * stacking drivers)
 */
struct bio {
	// 指向链表中下一个bio
	struct bio		*bi_next;	/* request queue link */
	// bio对应的磁盘
	struct block_device	*bi_bdev;
	blk_opf_t		bi_opf;		/* bottom bits REQ_OP, top bits
						 * req_flags.
						 */
	// 存储bio状态等信息
	unsigned short		bi_flags;	/* BIO_* below */
	unsigned short		bi_ioprio;
	blk_status_t		bi_status;
	atomic_t		__bi_remaining;
	// 存储操作数据在磁盘的位置
	struct bvec_iter	bi_iter;

	blk_qc_t		bi_cookie;
	bio_end_io_t		*bi_end_io;
	void			*bi_private;
#ifdef CONFIG_BLK_CGROUP
	/*
	 * Represents the association of the css and request_queue for the bio.
	 * If a bio goes direct to device, it will not have a blkg as it will
	 * not have a request_queue associated with it.  The reference is put
	 * on release of the bio.
	 */
	struct blkcg_gq		*bi_blkg;
	struct bio_issue	bi_issue;
#ifdef CONFIG_BLK_CGROUP_IOCOST
	u64			bi_iocost_cost;
#endif
#endif

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
	struct bio_crypt_ctx	*bi_crypt_context;
#endif

	union {
#if defined(CONFIG_BLK_DEV_INTEGRITY)
		struct bio_integrity_payload *bi_integrity; /* data integrity */
#endif
	};
	// bio对象包含bio_vec对象的数目
	unsigned short		bi_vcnt;	/* how many bio_vec's */

	/*
	 * Everything starting with bi_max_vecs will be preserved by bio_reset()
	 */
	// 最大的bio_vec数目
	unsigned short		bi_max_vecs;	/* max bvl_vecs we can hold */

	atomic_t		__bi_cnt;	/* pin count */

	// 存放段的数组，bio中每个段是由一个bio_vec的数据结构描述的
	struct bio_vec		*bi_io_vec;	/* the actual vec list */

	struct bio_set		*bi_pool;

	/*
	 * We can inline a number of vecs at the end of the bio, to avoid
	 * double allocations for a small number of bio_vecs. This member
	 * MUST obviously be kept at the very end of the bio.
	 */
	struct bio_vec		bi_inline_vecs[];
};
```

`struct request_queue`表示请求队列，每一个`gendisk`对象都有一个`request_queue`对象，保存对该`gendisk`对象的所有请求。
```c
// include/linux/blkdev.h
struct request_queue {
	struct request		*last_merge;
	// 指向I/O调度算法对象的指针
	struct elevator_queue	*elevator;

	struct percpu_ref	q_usage_counter;

	struct blk_queue_stats	*stats;
	struct rq_qos		*rq_qos;
	struct mutex		rq_qos_mutex;

	const struct blk_mq_ops	*mq_ops;

	/* sw queues */
	// 软件队列，软件队列的数量等于CPU核心数
	struct blk_mq_ctx __percpu	*queue_ctx;

	unsigned int		queue_depth;

	/* hw dispatch queues */
	// 硬件队列
	struct xarray		hctx_table;
	unsigned int		nr_hw_queues;

	/*
	 * The queue owner gets to use this for whatever they like.
	 * ll_rw_blk doesn't touch it.
	 */
	void			*queuedata;

	/*
	 * various queue flags, see QUEUE_* below
	 */
	unsigned long		queue_flags;
	/*
	 * Number of contexts that have called blk_set_pm_only(). If this
	 * counter is above zero then only RQF_PM requests are processed.
	 */
	atomic_t		pm_only;

	/*
	 * ida allocated id for this queue.  Used to index queues from
	 * ioctx.
	 */
	int			id;

	spinlock_t		queue_lock;

	struct gendisk		*disk;

	refcount_t		refs;

	/*
	 * mq queue kobject
	 */
	struct kobject *mq_kobj;

#ifdef  CONFIG_BLK_DEV_INTEGRITY
	struct blk_integrity integrity;
#endif	/* CONFIG_BLK_DEV_INTEGRITY */

#ifdef CONFIG_PM
	struct device		*dev;
	enum rpm_status		rpm_status;
#endif

	/*
	 * queue settings
	 */
	unsigned long		nr_requests;	/* Max # of requests */

	unsigned int		dma_pad_mask;

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
	struct blk_crypto_profile *crypto_profile;
	struct kobject *crypto_kobject;
#endif

	unsigned int		rq_timeout;

	struct timer_list	timeout;
	struct work_struct	timeout_work;

	atomic_t		nr_active_requests_shared_tags;

	struct blk_mq_tags	*sched_shared_tags;

	struct list_head	icq_list;
#ifdef CONFIG_BLK_CGROUP
	DECLARE_BITMAP		(blkcg_pols, BLKCG_MAX_POLS);
	struct blkcg_gq		*root_blkg;
	struct list_head	blkg_list;
	struct mutex		blkcg_mutex;
#endif

	struct queue_limits	limits;

	unsigned int		required_elevator_features;

	int			node;
#ifdef CONFIG_BLK_DEV_IO_TRACE
	struct blk_trace __rcu	*blk_trace;
#endif
	/*
	 * for flush operations
	 */
	struct blk_flush_queue	*fq;
	struct list_head	flush_list;

	struct list_head	requeue_list;
	spinlock_t		requeue_lock;
	struct delayed_work	requeue_work;

	struct mutex		sysfs_lock;
	struct mutex		sysfs_dir_lock;

	/*
	 * for reusing dead hctx instance in case of updating
	 * nr_hw_queues
	 */
	struct list_head	unused_hctx_list;
	spinlock_t		unused_hctx_lock;

	int			mq_freeze_depth;

#ifdef CONFIG_BLK_DEV_THROTTLING
	/* Throttle data */
	struct throtl_data *td;
#endif
	struct rcu_head		rcu_head;
	wait_queue_head_t	mq_freeze_wq;
	/*
	 * Protect concurrent access to q_usage_counter by
	 * percpu_ref_kill() and percpu_ref_reinit().
	 */
	struct mutex		mq_freeze_lock;

	int			quiesce_depth;

	struct blk_mq_tag_set	*tag_set;
	struct list_head	tag_set_list;

	struct dentry		*debugfs_dir;
	struct dentry		*sched_debugfs_dir;
	struct dentry		*rqos_debugfs_dir;
	/*
	 * Serializes all debugfs metadata operations using the above dentries.
	 */
	struct mutex		debugfs_mutex;

	bool			mq_sysfs_init_done;
};
```

`struct request`表示经过I/O调度之后的针对一个`gendisk`的请求，是`request_queue`的一个节点，一个request里面包含了一个或者多个bio，request存在的目的就是为了进行I/O的调度，通过request这个辅助结构，来给bio进行某种调度方法的排序，从而最大化地提高磁盘访问速度。
```c
// inlcude/linux/blkdev.h
struct request {
	// 这个请求所属的请求队列
	struct request_queue *q;
	// 指定这个请求将会发送到的软件队列
	struct blk_mq_ctx *mq_ctx;
	struct blk_mq_hw_ctx *mq_hctx;

	blk_opf_t cmd_flags;		/* op and common flags */
	req_flags_t rq_flags;

	int tag;
	int internal_tag;

	unsigned int timeout;

	/* the following two fields are internal, NEVER access directly */
	unsigned int __data_len;	/* total data len */
	sector_t __sector;		/* sector cursor */

	// 组成这个请求的bio链表的头指针
	struct bio *bio;
	// 组成这个请求的bio链表的尾指针
	struct bio *biotail;

	union {
		// 构建一个request的双向链表
		struct list_head queuelist;
		struct request *rq_next;
	};

	struct block_device *part;
#ifdef CONFIG_BLK_RQ_ALLOC_TIME
	/* Time that the first bio started allocating this request. */
	u64 alloc_time_ns;
#endif
	/* Time that this request was allocated for this IO. */
	u64 start_time_ns;
	/* Time that I/O was submitted to the device. */
	u64 io_start_time_ns;

#ifdef CONFIG_BLK_WBT
	unsigned short wbt_flags;
#endif
	/*
	 * rq sectors used for blk stats. It has the same value
	 * with blk_rq_sectors(rq), except that it never be zeroed
	 * by completion.
	 */
	unsigned short stats_sectors;

	/*
	 * Number of scatter-gather DMA addr+len pairs after
	 * physical address coalescing is performed.
	 */
	unsigned short nr_phys_segments;

#ifdef CONFIG_BLK_DEV_INTEGRITY
	unsigned short nr_integrity_segments;
#endif

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
	struct bio_crypt_ctx *crypt_ctx;
	struct blk_crypto_keyslot *crypt_keyslot;
#endif

	unsigned short ioprio;

	enum mq_rq_state state;
	atomic_t ref;

	unsigned long deadline;

	/*
	 * The hash is used inside the scheduler, and killed once the
	 * request reaches the dispatch list. The ipi_list is only used
	 * to queue the request for softirq completion, which is long
	 * after the request has been unhashed (and even removed from
	 * the dispatch list).
	 */
	union {
		struct hlist_node hash;	/* merge hash */
		struct llist_node ipi_list;
	};

	/*
	 * The rb_node is only used inside the io scheduler, requests
	 * are pruned when moved to the dispatch queue. special_vec must
	 * only be used if RQF_SPECIAL_PAYLOAD is set, and those cannot be
	 * insert into an IO scheduler.
	 */
	union {
		struct rb_node rb_node;	/* sort/lookup */
		struct bio_vec special_vec;
	};

	/*
	 * Three pointers are available for the IO schedulers, if they need
	 * more they have to dynamically allocate it.
	 */
	struct {
		struct io_cq		*icq;
		void			*priv[2];
	} elv;

	struct {
		unsigned int		seq;
		rq_end_io_fn		*saved_end_io;
	} flush;

	u64 fifo_time;

	/*
	 * completion callback.
	 */
	rq_end_io_fn *end_io;
	void *end_io_data;
};
```

### 2.4 blk-mq框架
多队列blk-mq框架如下：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240904140335.png)

blk-mq中使用了两层队列，将单个队列锁竞争分散到多个队列中，提高了block layer并发处理IO的能力：
+ 软件队列（对应 `struct blk_mq_ctx`）：blk-mq中为每个CPU核心分配一个软件队列（soft context dispatch queue），由于每个CPU有单独的队列，所以每个CPU上的这些I/O操作可以同时进行（同一个CPU中的进程间依然存在锁竞争的问题），而不存在锁竞争问题；
+ 硬件队列（对应 `struct blk_mq_hw_ctx`）：blk-mq为存储器件的每个硬件队列（目前多数存储器件只有1个）分配一个硬件派发队列（hard context dispatch queue），负责存放软件队列往这个硬件队列派发的I/O请求。在存储设备驱动初始化时，blk-mq会通过固定的映射关系将一个或多个软件队列映射（map）到一个硬件派发队列（同时保证映射到每个硬件队列的软件队列数量基本一致），之后这些软件队列上的I/O请求会往存储器件对应的硬件队列上派发。

基于blk-mq的块设备驱动初始化时，通过调用`blk_mq_init_queue()`初始化请求队列，其定义在`block/blk-mq.c`中

通过`submit_bio()`函数提交bio之后，会被`blk_mq_submit_bio()`处理，该函数定义在block/blk-mq.c文件中
```c
// block/blk-mq.c
/**
 * blk_mq_submit_bio - Create and send a request to block device.
 * @bio: Bio pointer.
 *
 * Builds up a request structure from @q and @bio and send to the device. The
 * request may not be queued directly to hardware if:
 * * This request can be merged with another one
 * * We want to place request at plug queue for possible future merging
 * * There is an IO scheduler active at this queue
 *
 * It will not queue the request if there is an error with the bio, or at the
 * request creation.
 */
void blk_mq_submit_bio(struct bio *bio)
{
	struct request_queue *q = bdev_get_queue(bio->bi_bdev);
	struct blk_plug *plug = blk_mq_plug(bio);
	const int is_sync = op_is_sync(bio->bi_opf);
	struct blk_mq_hw_ctx *hctx;
	struct request *rq = NULL;
	unsigned int nr_segs = 1;
	blk_status_t ret;

	bio = blk_queue_bounce(bio, q);
	if (bio_may_exceed_limits(bio, &q->limits)) {
		bio = __bio_split_to_limits(bio, &q->limits, &nr_segs);
		if (!bio)
			return;
	}

	bio_set_ioprio(bio);

	// 获取当前进程的request缓冲队列
	if (plug) {
		rq = rq_list_peek(&plug->cached_rq);
		if (rq && rq->q != q)
			rq = NULL;
	}
	if (rq) {
		// 尝试将bio合并到进程plug list的request，如果成功直接返回
		if (!bio_integrity_prep(bio))
			return;
		if (blk_mq_attempt_bio_merge(q, bio, nr_segs))
			return;
		if (blk_mq_can_use_cached_rq(rq, plug, bio))
			goto done;
		percpu_ref_get(&q->q_usage_counter);
	} else {
		if (unlikely(bio_queue_enter(bio)))
			return;
		if (!bio_integrity_prep(bio))
			goto fail;
	}
	// 判断I/O请求是否可以跟其它request合并，如果无法合并再将I/O请求转换为request进一步处理
	rq = blk_mq_get_new_requests(q, plug, bio, nr_segs);
	if (unlikely(!rq)) {
fail:
		blk_queue_exit(q);
		return;
	}

done:
	trace_block_getrq(bio);

	rq_qos_track(q, rq, bio);

	blk_mq_bio_to_request(rq, bio, nr_segs);

	ret = blk_crypto_rq_get_keyslot(rq);
	if (ret != BLK_STS_OK) {
		bio->bi_status = ret;
		bio_endio(bio);
		blk_mq_free_request(rq);
		return;
	}

	if (op_is_flush(bio->bi_opf) && blk_insert_flush(rq))
		return;

	if (plug) {
		blk_add_rq_to_plug(plug, rq);
		return;
	}

	hctx = rq->mq_hctx;
	if ((rq->rq_flags & RQF_USE_SCHED) ||
	    (hctx->dispatch_busy && (q->nr_hw_queues == 1 || !is_sync))) {
		blk_mq_insert_request(rq, 0);
		blk_mq_run_hw_queue(hctx, true);
	} else {
		// 调用blk_mq_try_issue_directly将request直接派发到块设备驱动
		blk_mq_run_dispatch_ops(q, blk_mq_try_issue_directly(hctx, rq));
	}
}
```


## 3. mq-deadline 调度算法

**`mq-deadline`调度器的特点：**
1. 对request划分了三个优先级（实时、普通、空闲）
2. 每个优先级将IO请求分为了read和write两种类型，对于每种类型的I/O分别有一颗红黑树和一个fifo队列，红黑树根据IO请求的起始扇区号进行排序，fifo队列记录了request进入调度器的顺序
3. read类型的request可以抢占write类型的request，但是会有一个饥饿计数防止write请求被“饿死”
4. 针对穿透性IO这种需要尽快发送到设备的IO设置另外一个dispatch队列，然后每次派发的时候都优先派发dispatch队列上的IO

### 3.1 源码分析
基于linux 6.6，源码位置: `block/mq-deadline.c`

`struct deadline_data`，一个块设备对应一个`struct deadline_data`，存储了调度器的状态、请求队列等信息，`struct dd_per_prio`针对每个优先级的请求维护了调度算法需要的队列数据及统计数据。
```c
// block/mq-deadline.c

/*
 * Deadline scheduler data per I/O priority (enum dd_prio). Requests are
 * present on both sort_list[] and fifo_list[].
 */
struct dd_per_prio {
	// 优先派发的IO队列
	struct list_head dispatch;
	// 针对读写请求分别维护了一颗红黑树和fifo队列，read下标为0，write下标为1
	struct rb_root sort_list[DD_DIR_COUNT];
	struct list_head fifo_list[DD_DIR_COUNT];
	/* Position of the most recently dispatched request. */
	// 
	sector_t latest_pos[DD_DIR_COUNT];
	// 该优先级请求的统计数据
	struct io_stats_per_prio stats;
};


struct deadline_data {
	/*
	 * run time data
	 */

	struct dd_per_prio per_prio[DD_PRIO_COUNT];

	/* Data direction of latest dispatched request. */
	enum dd_data_dir last_dir;
	unsigned int batching;		/* number of sequential requests made */
	// write饿的次数，不能超过writes_starved
	unsigned int starved;		/* times reads have starved writes */

	/*
	 * settings that change how the i/o scheduler behaves
	 */
	// 记录read和write的到期时间，默认的到期时间分别为：0.5s, 5s
	int fifo_expire[DD_DIR_COUNT];
	// 批量分发IO的最大个数
	int fifo_batch;
	// write饿的最大次数，默认设置为2
	int writes_starved;
	int front_merges;
	u32 async_depth;
	int prio_aging_expire;

	spinlock_t lock;
	spinlock_t zone_lock;
};
```

mq-deadline调度算法`struct evevator_type`定义
```c
// block/mq-deadline.c
static struct elevator_type mq_deadline = {
	// 调度器接口
	.ops = {
		.depth_updated		= dd_depth_updated,
		.limit_depth		= dd_limit_depth,
		// 插入request
		.insert_requests	= dd_insert_requests,
		// 分发request
		.dispatch_request	= dd_dispatch_request,
		.prepare_request	= dd_prepare_request,
		// request结束时调用
		.finish_request		= dd_finish_request,
		// 找到当前request的前一个request
		.next_request		= elv_rb_latter_request,
		// 找到当前request的后一个request
		.former_request		= elv_rb_former_request,
		// bio合并到mq-deadline的时候调用
		.bio_merge		= dd_bio_merge,
		// 找到一个可以将bio合并进去的request
		.request_merge		= dd_request_merge,
		// 两个request合并后调用
		.requests_merged	= dd_merged_requests,
		// bio合并到reques后调用
		.request_merged		= dd_request_merged,
		.has_work		= dd_has_work,
		// mq-deadline初始化
		.init_sched		= dd_init_sched,
		.exit_sched		= dd_exit_sched,
		.init_hctx		= dd_init_hctx,
	},

#ifdef CONFIG_BLK_DEBUG_FS
	.queue_debugfs_attrs = deadline_queue_debugfs_attrs,
#endif
	.elevator_attrs = deadline_attrs,
	.elevator_name = "mq-deadline",
	.elevator_alias = "deadline",
	.elevator_features = ELEVATOR_F_ZBD_SEQ_WRITE,
	.elevator_owner = THIS_MODULE,
};
MODULE_ALIAS("mq-deadline-iosched");
```

初始化mq-deadline
```c
/*
 * initialize elevator private data (deadline_data).
 */
static int dd_init_sched(struct request_queue *q, struct elevator_type *e)
{
	struct deadline_data *dd;
	struct elevator_queue *eq;
	enum dd_prio prio;
	int ret = -ENOMEM;

	eq = elevator_alloc(q, e);
	if (!eq)
		return ret;

	dd = kzalloc_node(sizeof(*dd), GFP_KERNEL, q->node);
	if (!dd)
		goto put_eq;

	eq->elevator_data = dd;

	// 初始化三种队列
	for (prio = 0; prio <= DD_PRIO_MAX; prio++) {
		struct dd_per_prio *per_prio = &dd->per_prio[prio];

		INIT_LIST_HEAD(&per_prio->dispatch);
		INIT_LIST_HEAD(&per_prio->fifo_list[DD_READ]);
		INIT_LIST_HEAD(&per_prio->fifo_list[DD_WRITE]);
		per_prio->sort_list[DD_READ] = RB_ROOT;
		per_prio->sort_list[DD_WRITE] = RB_ROOT;
	}
	// 设置超时时间
	dd->fifo_expire[DD_READ] = read_expire;
	dd->fifo_expire[DD_WRITE] = write_expire;
	dd->writes_starved = writes_starved;
	dd->front_merges = 1;
	dd->last_dir = DD_WRITE;
	dd->fifo_batch = fifo_batch;
	dd->prio_aging_expire = prio_aging_expire;
	spin_lock_init(&dd->lock);
	spin_lock_init(&dd->zone_lock);

	/* We dispatch from request queue wide instead of hw queue */
	blk_queue_flag_set(QUEUE_FLAG_SQ_SCHED, q);

	// request_queue关联该IO调度算法
	q->elevator = eq;
	return 0;

put_eq:
	kobject_put(&eq->kobj);
	return ret;
}
```

request插入到mq-deadline
```c
/*
 * add rq to rbtree and fifo
 */
static void dd_insert_request(struct blk_mq_hw_ctx *hctx, struct request *rq,
			      blk_insert_t flags, struct list_head *free)
{
	struct request_queue *q = hctx->queue;
	struct deadline_data *dd = q->elevator->elevator_data;
	const enum dd_data_dir data_dir = rq_data_dir(rq);
	u16 ioprio = req_get_ioprio(rq);
	u8 ioprio_class = IOPRIO_PRIO_CLASS(ioprio);
	struct dd_per_prio *per_prio;
	enum dd_prio prio;

	lockdep_assert_held(&dd->lock);

	/*
	 * This may be a requeue of a write request that has locked its
	 * target zone. If it is the case, this releases the zone lock.
	 */
	blk_req_zone_write_unlock(rq);

	prio = ioprio_class_to_prio[ioprio_class];
	per_prio = &dd->per_prio[prio];
	if (!rq->elv.priv[0]) {
		per_prio->stats.inserted++;
		rq->elv.priv[0] = (void *)(uintptr_t)1;
	}

	if (blk_mq_sched_try_insert_merge(q, rq, free))
		return;

	trace_block_rq_insert(rq);
	// 对于穿透型IO，直接插入到dispatch队列
	if (flags & BLK_MQ_INSERT_AT_HEAD) {
		list_add(&rq->queuelist, &per_prio->dispatch);
		rq->fifo_time = jiffies;
	} else {
		struct list_head *insert_before;

		deadline_add_rq_rb(per_prio, rq);

		if (rq_mergeable(rq)) {
			elv_rqhash_add(q, rq);
			if (!q->last_merge)
				q->last_merge = rq;
		}

		/*
		 * set expire time and add to fifo list
		 */
		// 设置request加入fifo队列的时间
		rq->fifo_time = jiffies + dd->fifo_expire[data_dir];
		insert_before = &per_prio->fifo_list[data_dir];
#ifdef CONFIG_BLK_DEV_ZONED
		/*
		 * Insert zoned writes such that requests are sorted by
		 * position per zone.
		 */
		if (blk_rq_is_seq_zoned_write(rq)) {
			struct request *rq2 = deadline_latter_request(rq);

			if (rq2 && blk_rq_zone_no(rq2) == blk_rq_zone_no(rq))
				insert_before = &rq2->queuelist;
		}
#endif
		// 加入到fifo队列
		list_add_tail(&rq->queuelist, insert_before);
	}
}
```

## 4. kyber 调度算法

## 5. bfq 调度算法

## 6. none调度算法
`none`调度器是一种不进行任何调度的模式，它通常用于设备不需要任何软件调度的时候（例如现代的SSD和NVME设备），这个调度器的概念类似于禁用I/O调度器，直接将请求交给设备，完全依赖于设备的固件进行调度

## 7. SD卡I/O调度算法对比测试
测试环境： E2000Q demo开发板，使用SD卡进行测试，linux 6.6内核

从sd卡块设备创建流程来看`mmc_blk_alloc_req()->device_add_disk()->elevator_init_mq()elevator_get_default()`，sd卡默认使用`mq-deadline`调度算法
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240905134944.png)

### 7.1 fio测试用例

1. 随机读
```bash
fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randread --iodepth=32 --numjobs=1
```

2. 随机写
```bash
fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randwrite --iodepth=32 --numjobs=1
```

3. 随机读写（70%读 30%写）
```bash
fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
```

### 7.2 mq-deadline调度算法测试结果

1. 随机读
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randread --iodepth=32 --numjobs=1
test: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=10.9MiB/s][r=2802 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2682: Thu Sep  5 09:54:33 2024
  read: IOPS=2491, BW=9965KiB/s (10.2MB/s)(584MiB/60012msec)
    slat (nsec): min=6120, max=57160, avg=10171.80, stdev=2349.93
    clat (usec): min=894, max=27946, avg=12831.60, stdev=4987.67
     lat (usec): min=903, max=27959, avg=12842.14, stdev=4987.83
    clat percentiles (usec):
     |  1.00th=[ 2474],  5.00th=[ 4555], 10.00th=[ 6194], 20.00th=[ 8455],
     | 30.00th=[10028], 40.00th=[11469], 50.00th=[12649], 60.00th=[14091],
     | 70.00th=[15401], 80.00th=[17171], 90.00th=[19530], 95.00th=[21365],
     | 99.00th=[24511], 99.50th=[25297], 99.90th=[26608], 99.95th=[27132],
     | 99.99th=[27657]
   bw (  KiB/s): min= 8760, max=11232, per=100.00%, avg=9964.51, stdev=1019.33, samples=120
   iops        : min= 2190, max= 2808, avg=2491.12, stdev=254.83, samples=120
  lat (usec)   : 1000=0.02%
  lat (msec)   : 2=0.50%, 4=2.91%, 10=25.86%, 20=62.50%, 50=8.21%
  cpu          : usr=1.09%, sys=3.27%, ctx=149486, majf=0, minf=53
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=149508,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=9965KiB/s (10.2MB/s), 9965KiB/s-9965KiB/s (10.2MB/s-10.2MB/s), io=584MiB (612MB), run=60012-60012msec

Disk stats (read/write):
  mmcblk1: ios=149128/0, merge=24/0, ticks=1913320/0, in_queue=1913320, util=99.87%
```

2. 随机写
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randwrite --iodepth=32 --numjobs=1
test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][w=2334KiB/s][w=583 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2697: Thu Sep  5 09:57:10 2024
  write: IOPS=806, BW=3227KiB/s (3304kB/s)(189MiB/60036msec); 0 zone resets
    slat (nsec): min=6480, max=91140, avg=9346.53, stdev=2082.12
    clat (msec): min=2, max=399, avg=39.65, stdev=28.59
     lat (msec): min=2, max=399, avg=39.66, stdev=28.59
    clat percentiles (msec):
     |  1.00th=[    8],  5.00th=[   14], 10.00th=[   18], 20.00th=[   25],
     | 30.00th=[   29], 40.00th=[   34], 50.00th=[   38], 60.00th=[   41],
     | 70.00th=[   45], 80.00th=[   51], 90.00th=[   58], 95.00th=[   64],
     | 99.00th=[  126], 99.50th=[  264], 99.90th=[  380], 99.95th=[  384],
     | 99.99th=[  393]
   bw (  KiB/s): min= 1176, max= 3568, per=100.00%, avg=3226.47, stdev=572.14, samples=120
   iops        : min=  294, max=  892, avg=806.62, stdev=143.03, samples=120
  lat (msec)   : 4=0.23%, 10=2.01%, 20=10.94%, 50=65.52%, 100=20.17%
  lat (msec)   : 250=0.59%, 500=0.55%
  cpu          : usr=0.34%, sys=0.93%, ctx=48424, majf=0, minf=21
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=99.9%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,48432,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
  WRITE: bw=3227KiB/s (3304kB/s), 3227KiB/s-3227KiB/s (3304kB/s-3304kB/s), io=189MiB (198MB), run=60036-60036msec

Disk stats (read/write):
  mmcblk1: ios=22/48290, merge=0/9, ticks=62/1914725, in_queue=1914787, util=99.96%
```

3. 随机读写
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
test: (g=0): rw=randrw, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [m(1)][100.0%][r=4884KiB/s,w=2044KiB/s][r=1221,w=511 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2640: Thu Sep  5 09:40:27 2024
  read: IOPS=1067, BW=4270KiB/s (4372kB/s)(250MiB/60020msec)
    slat (nsec): min=5460, max=94420, avg=9248.71, stdev=1913.72
    clat (usec): min=934, max=352019, avg=20528.39, stdev=13151.19
     lat (usec): min=943, max=352027, avg=20537.98, stdev=13151.22
    clat percentiles (msec):
     |  1.00th=[    4],  5.00th=[    7], 10.00th=[    9], 20.00th=[   13],
     | 30.00th=[   16], 40.00th=[   18], 50.00th=[   20], 60.00th=[   23],
     | 70.00th=[   25], 80.00th=[   28], 90.00th=[   32], 95.00th=[   35],
     | 99.00th=[   42], 99.50th=[   64], 99.90th=[  203], 99.95th=[  241],
     | 99.99th=[  347]
   bw (  KiB/s): min= 1512, max= 5384, per=100.00%, avg=4269.55, stdev=627.90, samples=120
   iops        : min=  378, max= 1346, avg=1067.38, stdev=156.97, samples=120
  write: IOPS=454, BW=1818KiB/s (1861kB/s)(107MiB/60020msec); 0 zone resets
    slat (nsec): min=5540, max=53260, avg=9557.98, stdev=2053.68
    clat (usec): min=1888, max=340231, avg=22159.15, stdev=13258.94
     lat (usec): min=1897, max=340242, avg=22169.05, stdev=13259.00
    clat percentiles (msec):
     |  1.00th=[    5],  5.00th=[    9], 10.00th=[   12], 20.00th=[   15],
     | 30.00th=[   18], 40.00th=[   20], 50.00th=[   22], 60.00th=[   24],
     | 70.00th=[   26], 80.00th=[   29], 90.00th=[   32], 95.00th=[   35],
     | 99.00th=[   47], 99.50th=[   70], 99.90th=[  220], 99.95th=[  239],
     | 99.99th=[  288]
   bw (  KiB/s): min=  656, max= 2184, per=100.00%, avg=1817.35, stdev=261.09, samples=120
   iops        : min=  164, max=  546, avg=454.33, stdev=65.29, samples=120
  lat (usec)   : 1000=0.01%
  lat (msec)   : 2=0.15%, 4=1.11%, 10=10.41%, 20=36.69%, 50=50.84%
  lat (msec)   : 100=0.48%, 250=0.28%, 500=0.04%
  cpu          : usr=0.62%, sys=1.83%, ctx=91334, majf=0, minf=25
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=64066,27272,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=4270KiB/s (4372kB/s), 4270KiB/s-4270KiB/s (4372kB/s-4372kB/s), io=250MiB (262MB), run=60020-60020msec
  WRITE: bw=1818KiB/s (1861kB/s), 1818KiB/s-1818KiB/s (1861kB/s-1861kB/s), io=107MiB (112MB), run=60020-60020msec

Disk stats (read/write):
  mmcblk1: ios=63940/27195, merge=3/1, ticks=1312121/602545, in_queue=1914666, util=99.98%
```

### 7.3 bfq调度算法测试结果
1. 随机读
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randread --iodepth=32 --numjobs=1
test: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=11.0MiB/s][r=2826 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2467: Thu Sep  5 09:06:23 2024
  read: IOPS=2572, BW=10.0MiB/s (10.5MB/s)(603MiB/60012msec)
    slat (usec): min=6, max=133, avg=11.01, stdev= 2.03
    clat (usec): min=893, max=30770, avg=12427.90, stdev=5038.70
     lat (usec): min=904, max=30780, avg=12439.24, stdev=5038.91
    clat percentiles (usec):
     |  1.00th=[ 2376],  5.00th=[ 4359], 10.00th=[ 5866], 20.00th=[ 7963],
     | 30.00th=[ 9634], 40.00th=[10945], 50.00th=[12256], 60.00th=[13566],
     | 70.00th=[15008], 80.00th=[16712], 90.00th=[19006], 95.00th=[21103],
     | 99.00th=[24773], 99.50th=[25822], 99.90th=[27132], 99.95th=[27657],
     | 99.99th=[27919]
   bw (  KiB/s): min= 8760, max=11392, per=99.98%, avg=10286.12, stdev=1142.24, samples=120
   iops        : min= 2190, max= 2848, avg=2571.51, stdev=285.55, samples=120
  lat (usec)   : 1000=0.03%
  lat (msec)   : 2=0.55%, 4=3.40%, 10=28.70%, 20=59.92%, 50=7.40%
  cpu          : usr=1.01%, sys=3.48%, ctx=154328, majf=0, minf=53
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=154351,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=10.0MiB/s (10.5MB/s), 10.0MiB/s-10.0MiB/s (10.5MB/s-10.5MB/s), io=603MiB (632MB), run=60012-60012msec

Disk stats (read/write):
  mmcblk1: ios=153972/0, merge=27/0, ticks=1913752/0, in_queue=1913752, util=99.87%
```

2. 随机写
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randwrite --iodepth=32 --numjobs=1
test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][w=3359KiB/s][w=839 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2481: Thu Sep  5 09:09:52 2024
  write: IOPS=807, BW=3229KiB/s (3307kB/s)(189MiB/60038msec); 0 zone resets
    slat (usec): min=6, max=120, avg=12.08, stdev= 3.22
    clat (msec): min=2, max=397, avg=39.62, stdev=28.61
     lat (msec): min=2, max=397, avg=39.63, stdev=28.61
    clat percentiles (msec):
     |  1.00th=[    7],  5.00th=[   14], 10.00th=[   18], 20.00th=[   25],
     | 30.00th=[   29], 40.00th=[   34], 50.00th=[   37], 60.00th=[   41],
     | 70.00th=[   46], 80.00th=[   52], 90.00th=[   58], 95.00th=[   65],
     | 99.00th=[  131], 99.50th=[  257], 99.90th=[  380], 99.95th=[  384],
     | 99.99th=[  393]
   bw (  KiB/s): min= 1160, max= 3568, per=100.00%, avg=3228.84, stdev=572.82, samples=120
   iops        : min=  290, max=  892, avg=807.20, stdev=143.20, samples=120
  lat (msec)   : 4=0.23%, 10=2.20%, 20=11.06%, 50=64.97%, 100=20.38%
  lat (msec)   : 250=0.64%, 500=0.53%
  cpu          : usr=0.32%, sys=1.23%, ctx=48761, majf=0, minf=21
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=99.9%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,48473,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
  WRITE: bw=3229KiB/s (3307kB/s), 3229KiB/s-3229KiB/s (3307kB/s-3307kB/s), io=189MiB (199MB), run=60038-60038msec

Disk stats (read/write):
  mmcblk1: ios=22/48335, merge=0/9, ticks=62/1914769, in_queue=1914831, util=99.96%
```

3. 随机读写
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
test: (g=0): rw=randrw, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [m(1)][100.0%][r=4356KiB/s,w=1753KiB/s][r=1089,w=438 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2656: Thu Sep  5 09:42:36 2024
  read: IOPS=957, BW=3829KiB/s (3920kB/s)(224MiB/60022msec)
    slat (usec): min=7, max=121, avg=10.70, stdev= 1.53
    clat (usec): min=912, max=251305, avg=23168.60, stdev=13624.92
     lat (usec): min=922, max=251316, avg=23179.60, stdev=13624.90
    clat percentiles (msec):
     |  1.00th=[    4],  5.00th=[    8], 10.00th=[   11], 20.00th=[   15],
     | 30.00th=[   18], 40.00th=[   20], 50.00th=[   23], 60.00th=[   25],
     | 70.00th=[   28], 80.00th=[   31], 90.00th=[   35], 95.00th=[   40],
     | 99.00th=[   57], 99.50th=[   97], 99.90th=[  186], 99.95th=[  205],
     | 99.99th=[  239]
   bw (  KiB/s): min= 1344, max= 4488, per=100.00%, avg=3828.26, stdev=537.11, samples=120
   iops        : min=  336, max= 1122, avg=957.06, stdev=134.27, samples=120
  write: IOPS=409, BW=1638KiB/s (1677kB/s)(95.0MiB/60022msec); 0 zone resets
    slat (nsec): min=7680, max=60680, avg=11098.48, stdev=1668.39
    clat (usec): min=1524, max=237188, avg=23949.33, stdev=13273.47
     lat (usec): min=1536, max=237199, avg=23960.73, stdev=13273.43
    clat percentiles (msec):
     |  1.00th=[    5],  5.00th=[    9], 10.00th=[   11], 20.00th=[   15],
     | 30.00th=[   18], 40.00th=[   21], 50.00th=[   24], 60.00th=[   26],
     | 70.00th=[   29], 80.00th=[   32], 90.00th=[   36], 95.00th=[   40],
     | 99.00th=[   55], 99.50th=[   93], 99.90th=[  174], 99.95th=[  203],
     | 99.99th=[  230]
   bw (  KiB/s): min=  640, max= 1904, per=100.00%, avg=1637.53, stdev=221.25, samples=120
   iops        : min=  160, max=  476, avg=409.38, stdev=55.31, samples=120
  lat (usec)   : 1000=0.01%
  lat (msec)   : 2=0.15%, 4=0.85%, 10=8.35%, 20=30.35%, 50=59.19%
  lat (msec)   : 100=0.64%, 250=0.47%, 500=0.01%
  cpu          : usr=0.57%, sys=1.75%, ctx=82017, majf=0, minf=25
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=57449,24575,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=3829KiB/s (3920kB/s), 3829KiB/s-3829KiB/s (3920kB/s-3920kB/s), io=224MiB (235MB), run=60022-60022msec
  WRITE: bw=1638KiB/s (1677kB/s), 1638KiB/s-1638KiB/s (1677kB/s-1677kB/s), io=95.0MiB (101MB), run=60022-60022msec

Disk stats (read/write):
  mmcblk1: ios=57322/24512, merge=7/1, ticks=1327614/587093, in_queue=1914707, util=99.97%
```

### 7.4 none调度算法测试结果
1. 随机读
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randread --iodepth=32 --numjobs=1
test: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=10.9MiB/s][r=2803 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2528: Thu Sep  5 09:12:19 2024
  read: IOPS=2544, BW=9.94MiB/s (10.4MB/s)(596MiB/60001msec)
    slat (usec): min=61, max=3663, avg=387.18, stdev=62.75
    clat (usec): min=339, max=17292, avg=12183.67, stdev=1411.06
     lat (usec): min=673, max=17752, avg=12571.49, stdev=1455.79
    clat percentiles (usec):
     |  1.00th=[10552],  5.00th=[10683], 10.00th=[10814], 20.00th=[10945],
     | 30.00th=[11076], 40.00th=[11207], 50.00th=[11338], 60.00th=[11600],
     | 70.00th=[13960], 80.00th=[14091], 90.00th=[14091], 95.00th=[14222],
     | 99.00th=[14222], 99.50th=[14222], 99.90th=[14353], 99.95th=[14353],
     | 99.99th=[14353]
   bw (  KiB/s): min= 8502, max=11304, per=99.89%, avg=10166.07, stdev=1127.27, samples=119
   iops        : min= 2125, max= 2826, avg=2541.50, stdev=281.83, samples=119
  lat (usec)   : 500=0.01%, 750=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=99.98%
  cpu          : usr=1.65%, sys=4.37%, ctx=152665, majf=0, minf=51
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=152665,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=9.94MiB/s (10.4MB/s), 9.94MiB/s-9.94MiB/s (10.4MB/s-10.4MB/s), io=596MiB (625MB), run=60001-60001msec

Disk stats (read/write):
  mmcblk1: ios=152346/0, merge=0/0, ticks=117466/0, in_queue=117466, util=99.87%
```

2. 随机写
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k --rw=randwrite --iodepth=32 --numjobs=1
test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][w=2828KiB/s][w=707 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2532: Thu Sep  5 09:14:24 2024
  write: IOPS=806, BW=3225KiB/s (3302kB/s)(189MiB/60020msec); 0 zone resets
    slat (usec): min=47, max=52208, avg=1236.46, stdev=1630.13
    clat (msec): min=12, max=369, avg=38.44, stdev=23.46
     lat (msec): min=24, max=370, avg=39.68, stdev=23.96
    clat percentiles (msec):
     |  1.00th=[   33],  5.00th=[   34], 10.00th=[   34], 20.00th=[   34],
     | 30.00th=[   35], 40.00th=[   36], 50.00th=[   37], 60.00th=[   37],
     | 70.00th=[   37], 80.00th=[   37], 90.00th=[   38], 95.00th=[   40],
     | 99.00th=[  124], 99.50th=[  249], 99.90th=[  363], 99.95th=[  363],
     | 99.99th=[  368]
   bw (  KiB/s): min= 1152, max= 3552, per=99.99%, avg=3223.54, stdev=565.70, samples=120
   iops        : min=  288, max=  888, avg=805.88, stdev=141.43, samples=120
  lat (msec)   : 20=0.01%, 50=97.46%, 100=1.46%, 250=0.58%, 500=0.49%
  cpu          : usr=0.28%, sys=81.30%, ctx=96826, majf=0, minf=22
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=99.9%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,48386,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
  WRITE: bw=3225KiB/s (3302kB/s), 3225KiB/s-3225KiB/s (3302kB/s-3302kB/s), io=189MiB (198MB), run=60020-60020msec

Disk stats (read/write):
  mmcblk1: ios=27/48375, merge=0/0, ticks=64/119276, in_queue=119340, util=99.98%
```

3. 随机读写
```bash
root@Ubuntu:~# fio --name=test --filename=/dev/mmcblk1p1 --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
test: (g=0): rw=randrw, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [m(1)][100.0%][r=4296KiB/s,w=1761KiB/s][r=1074,w=440 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=2672: Thu Sep  5 09:47:13 2024
  read: IOPS=959, BW=3838KiB/s (3931kB/s)(225MiB/60003msec)
    slat (usec): min=86, max=48115, avg=725.54, stdev=960.81
    clat (usec): min=493, max=228303, avg=22335.16, stdev=9004.58
     lat (msec): min=4, max=228, avg=23.06, stdev= 9.26
    clat percentiles (msec):
     |  1.00th=[   16],  5.00th=[   17], 10.00th=[   18], 20.00th=[   20],
     | 30.00th=[   21], 40.00th=[   22], 50.00th=[   23], 60.00th=[   23],
     | 70.00th=[   24], 80.00th=[   24], 90.00th=[   25], 95.00th=[   26],
     | 99.00th=[   32], 99.50th=[   69], 99.90th=[  163], 99.95th=[  182],
     | 99.99th=[  209]
   bw (  KiB/s): min= 1424, max= 4616, per=99.97%, avg=3836.99, stdev=477.40, samples=120
   iops        : min=  356, max= 1154, avg=959.24, stdev=119.34, samples=120
  write: IOPS=410, BW=1641KiB/s (1681kB/s)(96.2MiB/60003msec); 0 zone resets
    slat (usec): min=251, max=46481, avg=727.67, stdev=963.60
    clat (msec): min=4, max=228, avg=23.31, stdev= 9.96
     lat (msec): min=4, max=233, avg=24.03, stdev=10.15
    clat percentiles (msec):
     |  1.00th=[   16],  5.00th=[   18], 10.00th=[   19], 20.00th=[   21],
     | 30.00th=[   22], 40.00th=[   23], 50.00th=[   23], 60.00th=[   24],
     | 70.00th=[   24], 80.00th=[   25], 90.00th=[   26], 95.00th=[   27],
     | 99.00th=[   55], 99.50th=[   96], 99.90th=[  174], 99.95th=[  197],
     | 99.99th=[  209]
   bw (  KiB/s): min=  664, max= 1936, per=99.98%, avg=1640.73, stdev=201.56, samples=120
   iops        : min=  166, max=  484, avg=410.18, stdev=50.39, samples=120
  lat (usec)   : 500=0.01%
  lat (msec)   : 10=0.01%, 20=24.48%, 50=74.52%, 100=0.57%, 250=0.42%
  cpu          : usr=0.63%, sys=46.41%, ctx=106864, majf=0, minf=25
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=57580,24621,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=3838KiB/s (3931kB/s), 3838KiB/s-3838KiB/s (3931kB/s-3931kB/s), io=225MiB (236MB), run=60003-60003msec
  WRITE: bw=1641KiB/s (1681kB/s), 1641KiB/s-1641KiB/s (1681kB/s-1681kB/s), io=96.2MiB (101MB), run=60003-60003msec

Disk stats (read/write):
  mmcblk1: ios=57464/24573, merge=0/0, ticks=68497/50539, in_queue=119036, util=99.96%
```

### 7.5 测试结果分析

**随机读测试**
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240906170539.png)

**随机写测试**
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240906170559.png)

**随机读写测试**
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240906170630.png)

1. 在单独读或者单独写的场景下，三种调度算法的I/O吞吐能力基本上没什么区别
2. 在混合读写的场景下，`mq-deadline`的I/O吞吐能力最强，`bfq`和`none`I/O吞吐能力相近
3. 使用`none`调度算法的I/O延迟是最小的，`mq-deadline`和`bfq`的I/O延迟相近

从测试结果来看，SD卡存储介质上做系统盘，使用mq-deadline调度算法可以带来更高的I/O吞吐能力


