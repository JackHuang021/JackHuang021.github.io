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
3. I/O调度层对`request_queue`中的`request`进行预处理后，调用请求队列的回调函数对队列中的`request`进行处理
4. 请求队列的回调函数由块设备驱动层提供，块设备驱动层会从请求队列提取`request`并进行实质性的处理

### 2.3 重要的结构体

#### 2.3.1 struct bio
`struct bio`，用来描述单一的I/O请求，记录了一次I/O操作所必需的相关信息，
```c
// include/linux/blk_types.h

/*
 * bio flags
 */
enum {
	BIO_PAGE_PINNED,	/* Unpin pages in bio_release_pages() */
	BIO_CLONED,		/* doesn't own data */
	BIO_BOUNCED,		/* bio is a bounce bio */
	BIO_QUIET,		/* Make BIO Quiet */
	BIO_CHAIN,		/* chained bio, ->bi_remaining in effect */
	BIO_REFFED,		/* bio has elevated ->bi_cnt */
	BIO_BPS_THROTTLED,	/* This bio has already been subjected to
				 * throttling rules. Don't do it again. */
	BIO_TRACE_COMPLETION,	/* bio_endio() should trace the final completion
				 * of this bio. */
	BIO_CGROUP_ACCT,	/* has been accounted to a cgroup */
	BIO_QOS_THROTTLED,	/* bio went through rq_qos throttle path */
	BIO_QOS_MERGED,		/* but went through rq_qos merge path */
	BIO_REMAPPED,
	BIO_ZONE_WRITE_LOCKED,	/* Owns a zoned device zone write lock */
	BIO_FLAG_LAST
};

/*
 * main unit of I/O for the block layer and lower layers (ie drivers and
 * stacking drivers)
 */
struct bio {
	// 指向链表中下一个bio
	struct bio		*bi_next;	/* request queue link */
	// bio对应的磁盘
	struct block_device	*bi_bdev;
	// 低24位为请求标志位，高8位为请求操作位
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

请求操作以及请求标志定义在`include/linux/blk_types.h`
```c
// include/linux/blk_types.h
#define REQ_OP_BITS	8
#define REQ_OP_MASK	(__force blk_opf_t)((1 << REQ_OP_BITS) - 1)
#define REQ_FLAG_BITS	24

/**
 * enum req_op - Operations common to the bio and request structures.
 * We use 8 bits for encoding the operation, and the remaining 24 for flags.
 *
 * The least significant bit of the operation number indicates the data
 * transfer direction:
 *
 *   - if the least significant bit is set transfers are TO the device
 *   - if the least significant bit is not set transfers are FROM the device
 *
 * If a operation does not transfer data the least significant bit has no
 * meaning.
 */
enum req_op {
	/* read sectors from the device */
	REQ_OP_READ		= (__force blk_opf_t)0,
	/* write sectors to the device */
	REQ_OP_WRITE		= (__force blk_opf_t)1,
	/* flush the volatile write cache */
	REQ_OP_FLUSH		= (__force blk_opf_t)2,
	/* discard sectors */
	REQ_OP_DISCARD		= (__force blk_opf_t)3,
	/* securely erase sectors */
	REQ_OP_SECURE_ERASE	= (__force blk_opf_t)5,
	/* write the zero filled sector many times */
	REQ_OP_WRITE_ZEROES	= (__force blk_opf_t)9,
	/* Open a zone */
	REQ_OP_ZONE_OPEN	= (__force blk_opf_t)10,
	/* Close a zone */
	REQ_OP_ZONE_CLOSE	= (__force blk_opf_t)11,
	/* Transition a zone to full */
	REQ_OP_ZONE_FINISH	= (__force blk_opf_t)12,
	/* write data at the current zone write pointer */
	REQ_OP_ZONE_APPEND	= (__force blk_opf_t)13,
	/* reset a zone write pointer */
	REQ_OP_ZONE_RESET	= (__force blk_opf_t)15,
	/* reset all the zone present on the device */
	REQ_OP_ZONE_RESET_ALL	= (__force blk_opf_t)17,

	/* Driver private requests */
	REQ_OP_DRV_IN		= (__force blk_opf_t)34,
	REQ_OP_DRV_OUT		= (__force blk_opf_t)35,

	REQ_OP_LAST		= (__force blk_opf_t)36,
};

enum req_flag_bits {
	__REQ_FAILFAST_DEV =	/* no driver retries of device errors */
		REQ_OP_BITS,
	__REQ_FAILFAST_TRANSPORT, /* no driver retries of transport errors */
	__REQ_FAILFAST_DRIVER,	/* no driver retries of driver errors */
	__REQ_SYNC,		/* request is sync (sync write or read) */
	__REQ_META,		/* metadata io request */
	__REQ_PRIO,		/* boost priority in cfq */
	__REQ_NOMERGE,		/* don't touch this for merging */
	__REQ_IDLE,		/* anticipate more IO after this one */
	__REQ_INTEGRITY,	/* I/O includes block integrity payload */
	__REQ_FUA,		/* forced unit access */
	__REQ_PREFLUSH,		/* request for cache flush */
	__REQ_RAHEAD,		/* read ahead, can fail anytime */
	__REQ_BACKGROUND,	/* background IO */
	__REQ_NOWAIT,           /* Don't wait if request will block */
	__REQ_POLLED,		/* caller polls for completion using bio_poll */
	__REQ_ALLOC_CACHE,	/* allocate IO from cache if available */
	__REQ_SWAP,		/* swap I/O */
	__REQ_DRV,		/* for driver use */
	__REQ_FS_PRIVATE,	/* for file system (submitter) use */

	/*
	 * Command specific flags, keep last:
	 */
	/* for REQ_OP_WRITE_ZEROES: */
	__REQ_NOUNMAP,		/* do not free blocks when zeroing */

	__REQ_NR_BITS,		/* stops here */
};

#define REQ_FAILFAST_DEV	\
			(__force blk_opf_t)(1ULL << __REQ_FAILFAST_DEV)
#define REQ_FAILFAST_TRANSPORT	\
			(__force blk_opf_t)(1ULL << __REQ_FAILFAST_TRANSPORT)
#define REQ_FAILFAST_DRIVER	\
			(__force blk_opf_t)(1ULL << __REQ_FAILFAST_DRIVER)
#define REQ_SYNC	(__force blk_opf_t)(1ULL << __REQ_SYNC)
#define REQ_META	(__force blk_opf_t)(1ULL << __REQ_META)
#define REQ_PRIO	(__force blk_opf_t)(1ULL << __REQ_PRIO)
#define REQ_NOMERGE	(__force blk_opf_t)(1ULL << __REQ_NOMERGE)
#define REQ_IDLE	(__force blk_opf_t)(1ULL << __REQ_IDLE)
#define REQ_INTEGRITY	(__force blk_opf_t)(1ULL << __REQ_INTEGRITY)
#define REQ_FUA		(__force blk_opf_t)(1ULL << __REQ_FUA)
#define REQ_PREFLUSH	(__force blk_opf_t)(1ULL << __REQ_PREFLUSH)
#define REQ_RAHEAD	(__force blk_opf_t)(1ULL << __REQ_RAHEAD)
#define REQ_BACKGROUND	(__force blk_opf_t)(1ULL << __REQ_BACKGROUND)
#define REQ_NOWAIT	(__force blk_opf_t)(1ULL << __REQ_NOWAIT)
#define REQ_POLLED	(__force blk_opf_t)(1ULL << __REQ_POLLED)
#define REQ_ALLOC_CACHE	(__force blk_opf_t)(1ULL << __REQ_ALLOC_CACHE)
#define REQ_SWAP	(__force blk_opf_t)(1ULL << __REQ_SWAP)
#define REQ_DRV		(__force blk_opf_t)(1ULL << __REQ_DRV)
#define REQ_FS_PRIVATE	(__force blk_opf_t)(1ULL << __REQ_FS_PRIVATE)

#define REQ_NOUNMAP	(__force blk_opf_t)(1ULL << __REQ_NOUNMAP)

#define REQ_FAILFAST_MASK \
	(REQ_FAILFAST_DEV | REQ_FAILFAST_TRANSPORT | REQ_FAILFAST_DRIVER)

#define REQ_NOMERGE_FLAGS \
	(REQ_NOMERGE | REQ_PREFLUSH | REQ_FUA)
```

#### 2.3.2 struct bio_vec
`struct bio_vec`描述指定page中的一块连续的区域，在bio中描述的就是一个page中的一个段(segment)，bio段就是描述所有读或者写的数据在内存中的位置
```c
// include/linux/bvec.h
/**
 * struct bio_vec - a contiguous range of physical memory addresses
 * @bv_page:   First page associated with the address range.
 * @bv_len:    Number of bytes in the address range.
 * @bv_offset: Start of the address range relative to the start of @bv_page.
 *
 * The following holds for a bvec if n * PAGE_SIZE < bv_offset + bv_len:
 *
 *   nth_page(@bv_page, n) == @bv_page + n
 *
 * This holds because page_is_mergeable() checks the above property.
 */
struct bio_vec {
	// 指向段所在的page
	struct page	*bv_page;
	// 段的字节长度
	unsigned int	bv_len;
	// 段在page中的偏移
	unsigned int	bv_offset;
};
```
#### 2.3.3 struct bvec_iter
`struct bvec_iter`用于记录当前`bio_vec`被处理的情况
```c
// include/linux/bvec.h
struct bvec_iter {
	// IO请求的块设备起始扇区，每个扇区512字节
	sector_t		bi_sector;	/* device address in 512 byte
						   sectors */
	// 待传输的字节大小
	unsigned int		bi_size;	/* residual I/O count */
	// 遍历bio_vec的索引
	unsigned int		bi_idx;		/* current index into bvl_vec */
	// 当前bio_vec中已经处理完成的字节数
	unsigned int            bi_bvec_done;	/* number of bytes completed in
						   current bvec */
} __packed;
```

#### 2.3.4 struct request_queue
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

#### 2.3.5 struct request
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

#### 2.3.6 struct gendisk
linux内核使用gendisk结构体来描述一个磁盘设备，用来存储该设备的硬盘信息，包括请求队列，分区链表和块设备操作函数等
```c
// include/linux/blkdev.h
struct gendisk {
	/*
	 * major/first_minor/minors should not be set by any new driver, the
	 * block core will take care of allocating them automatically.
	 */
	// 通过register_blkdev函数申请的主设备号
	int major;
	int first_minor;
	// 磁盘的次设备号数量，也就是磁盘的分区数量
	int minors;
	// 磁盘名，磁盘设备注册到内核后，可以在/dev下看到名为disk_name的块设备
	char disk_name[DISK_NAME_LEN];	/* name of major driver */

	unsigned short events;		/* supported events */
	unsigned short event_flags;	/* flags related to event processing */

	struct xarray part_tbl;
	struct block_device *part0;

	const struct block_device_operations *fops;
	struct request_queue *queue;
	void *private_data;

	struct bio_set bio_split;

	int flags;
	// 磁盘的状态标识
	unsigned long state;
#define GD_NEED_PART_SCAN		0
#define GD_READ_ONLY			1
#define GD_DEAD				2
#define GD_NATIVE_CAPACITY		3
#define GD_ADDED			4
#define GD_SUPPRESS_PART_SCAN		5
#define GD_OWNS_QUEUE			6

	struct mutex open_mutex;	/* open/close mutex */
	unsigned open_partitions;	/* number of open partitions */

	struct backing_dev_info	*bdi;
	struct kobject queue_kobj;	/* the queue/ directory */
	struct kobject *slave_dir;
#ifdef CONFIG_BLOCK_HOLDER_DEPRECATED
	struct list_head slave_bdevs;
#endif
	struct timer_rand_state *random;
	atomic_t sync_io;		/* RAID */
	struct disk_events *ev;

#ifdef CONFIG_BLK_DEV_ZONED
	/*
	 * Zoned block device information for request dispatch control.
	 * nr_zones is the total number of zones of the device. This is always
	 * 0 for regular block devices. conv_zones_bitmap is a bitmap of nr_zones
	 * bits which indicates if a zone is conventional (bit set) or
	 * sequential (bit clear). seq_zones_wlock is a bitmap of nr_zones
	 * bits which indicates if a zone is write locked, that is, if a write
	 * request targeting the zone was dispatched.
	 *
	 * Reads of this information must be protected with blk_queue_enter() /
	 * blk_queue_exit(). Modifying this information is only allowed while
	 * no requests are being processed. See also blk_mq_freeze_queue() and
	 * blk_mq_unfreeze_queue().
	 */
	unsigned int		nr_zones;
	unsigned int		max_open_zones;
	unsigned int		max_active_zones;
	unsigned long		*conv_zones_bitmap;
	unsigned long		*seq_zones_wlock;
#endif /* CONFIG_BLK_DEV_ZONED */

#if IS_ENABLED(CONFIG_CDROM)
	struct cdrom_device_info *cdi;
#endif
	int node_id;
	struct badblocks *bb;
	struct lockdep_map lockdep_map;
	u64 diskseq;
	blk_mode_t open_mode;

	/*
	 * Independent sector access ranges. This is always NULL for
	 * devices that do not have multiple independent access ranges.
	 */
	struct blk_independent_access_ranges *ia_ranges;
};
```

#### 2.3.7 struct block_device
`struct block_device`对块设备的进一步封装
```c
// include/linux/blk_types.h
struct block_device {
	sector_t		bd_start_sect;
	// 块设备最大的扇区数量
	sector_t		bd_nr_sectors;
	// 指向gendisk
	struct gendisk *	bd_disk;
	// 指向对应的请求队列
	struct request_queue *	bd_queue;
	struct disk_stats __percpu *bd_stats;
	unsigned long		bd_stamp;
	// 是否为只读设备
	bool			bd_read_only;	/* read-only policy */
	u8			bd_partno;
	bool			bd_write_holder;
	bool			bd_has_submit_bio;
	dev_t			bd_dev;
	atomic_t		bd_openers;
	spinlock_t		bd_size_lock; /* for bd_inode->i_size updates */
	struct inode *		bd_inode;	/* will die */
	void *			bd_claiming;
	void *			bd_holder;
	const struct blk_holder_ops *bd_holder_ops;
	struct mutex		bd_holder_lock;
	/* The counter of freeze processes */
	int			bd_fsfreeze_count;
	int			bd_holders;
	struct kobject		*bd_holder_dir;

	/* Mutex for freeze */
	struct mutex		bd_fsfreeze_mutex;
	struct super_block	*bd_fsfreeze_sb;

	struct partition_meta_info *bd_meta_info;
#ifdef CONFIG_FAIL_MAKE_REQUEST
	bool			bd_make_it_fail;
#endif
	/*
	 * keep this out-of-line as it's both big and not needed in the fast
	 * path
	 */
	struct device		bd_device;
} __randomize_layout;
```

## 3 blk-mq框架
多队列blk-mq框架如下：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240904140335.png)

blk-mq中使用了两层队列，将单个队列锁竞争分散到多个队列中，提高了block layer并发处理IO的能力：
+ 软件队列（对应 `struct blk_mq_ctx`）：blk-mq中为每个CPU核心分配一个软件队列（soft context dispatch queue），由于每个CPU有单独的队列，所以每个CPU上的这些I/O操作可以同时进行（同一个CPU中的进程间依然存在锁竞争的问题），而不存在锁竞争问题；
+ 硬件队列（对应 `struct blk_mq_hw_ctx`）：blk-mq为存储器件的每个硬件队列（目前多数存储器件只有1个）分配一个硬件派发队列（hard context dispatch queue），负责存放软件队列往这个硬件队列派发的I/O请求。在存储设备驱动初始化时，blk-mq会通过固定的映射关系将一个或多个软件队列映射（map）到一个硬件派发队列（同时保证映射到每个硬件队列的软件队列数量基本一致），之后这些软件队列上的I/O请求会往存储器件对应的硬件队列上派发。

基于blk-mq的块设备驱动初始化时，通过调用`blk_mq_init_queue()`初始化请求队列，其定义在`block/blk-mq.c`中

### 3.1 blk-mq相关的数据结构

#### 3.1.1 struct blk_mq_tag_set
`struct blk_mq_tag_set`包含了一个新的块设备向block layer注册的所有重要信息，抽象了存储设备的IO特征
```c
// include/linux.blk-mq.h
/**
 * struct blk_mq_tag_set - tag set that can be shared between request queues
 * @ops:	   Pointers to functions that implement block driver behavior.
 * @map:	   One or more ctx -> hctx mappings. One map exists for each
 *		   hardware queue type (enum hctx_type) that the driver wishes
 *		   to support. There are no restrictions on maps being of the
 *		   same size, and it's perfectly legal to share maps between
 *		   types.
 * @nr_maps:	   Number of elements in the @map array. A number in the range
 *		   [1, HCTX_MAX_TYPES].
 * @nr_hw_queues:  Number of hardware queues supported by the block driver that
 *		   owns this data structure.
 * @queue_depth:   Number of tags per hardware queue, reserved tags included.
 * @reserved_tags: Number of tags to set aside for BLK_MQ_REQ_RESERVED tag
 *		   allocations.
 * @cmd_size:	   Number of additional bytes to allocate per request. The block
 *		   driver owns these additional bytes.
 * @numa_node:	   NUMA node the storage adapter has been connected to.
 * @timeout:	   Request processing timeout in jiffies.
 * @flags:	   Zero or more BLK_MQ_F_* flags.
 * @driver_data:   Pointer to data owned by the block driver that created this
 *		   tag set.
 * @tags:	   Tag sets. One tag set per hardware queue. Has @nr_hw_queues
 *		   elements.
 * @shared_tags:
 *		   Shared set of tags. Has @nr_hw_queues elements. If set,
 *		   shared by all @tags.
 * @tag_list_lock: Serializes tag_list accesses.
 * @tag_list:	   List of the request queues that use this tag set. See also
 *		   request_queue.tag_set_list.
 * @srcu:	   Use as lock when type of the request queue is blocking
 *		   (BLK_MQ_F_BLOCKING).
 */
struct blk_mq_tag_set {
	const struct blk_mq_ops	*ops;
	// 用于保存软件队列到硬件队列的映射表
	struct blk_mq_queue_map	map[HCTX_MAX_TYPES];
	// map中元素的数量，他的范围在[1，HCTX_MAX_TYPES]之间
	unsigned int		nr_maps;
	// 块设备硬件队列的数量
	unsigned int		nr_hw_queues;
	// 硬件队列的深度，包含预留的reserved_tags个数
	unsigned int		queue_depth;
	// 每个硬件队列预留的元素个数
	unsigned int		reserved_tags;
	// 块设备驱动为每个request分配的额外的空间大小，一般用于存放设备驱动payload数据
	unsigned int		cmd_size;
	// numa节点，分配request内存时使用，避免远程内存访问问题
	int			numa_node;
	unsigned int		timeout;
	unsigned int		flags;
	void			*driver_data;
	struct blk_mq_tags	**tags;

	struct blk_mq_tags	*shared_tags;

	struct mutex		tag_list_lock;
	// 用于将blk_mq_tag_set组成链表，便于管理
	struct list_head	tag_list;
	struct srcu_struct	*srcu;
};
```

#### 3.1.2 struct blk_mq_tags
`struct blk_mq_tags`，用于管理`struct request`的分配，`blk_mq_tags`与硬件队列`blk_mq_hw_ctx`一一对应，tag是用来为request打标签的，只有一个request被分配了一个tag，这个request才能进行真正的IO传输

每当一个bio被提交，如果被转换成request的话，需要进行如下步骤：
+ 首先从bitmap_tags或或者breserved_tags分配一个tag；
+ 然后根据tag索引，获取static_rqs[tag]作为当前的request，并初始化该请求成员；
+ 设置rqs[tag]=static_rqs[tag]；

```c
// include/linux/blk-mq.h
/*
 * Tag address space map.
 */
struct blk_mq_tags {
	// 每个硬件队列的深度（包含预留的个数reserved_tags）
	unsigned int nr_tags;
	// 每个硬件队列预留的tag个数
	unsigned int nr_reserved_tags;
	// 活跃的request数量
	unsigned int active_queues;
	// tag的位图；每个bit代表一个tag标记，用于标示硬件队列中的request；1位已分配，0为为分配
	struct sbitmap_queue bitmap_tags;
	// 用于管理reserved的tag
	struct sbitmap_queue breserved_tags;

	// request长度为nr_tags，在初始化时会按照硬件队列的深度来分配static_rqs
	struct request **rqs;
	struct request **static_rqs;
	struct list_head page_list;

	/*
	 * used to clear request reference in rqs[] before freeing one
	 * request pool
	 */
	spinlock_t lock;
};
```

#### 3.1.3 struct blk_mq_ctx
`struct blk_mq_ctx`，用来表示软件队列，软件队列个数与CPU的数量相同
```c
// block/blk-mq.h
/**
 * struct blk_mq_ctx - State for a software queue facing the submitting CPUs
 */
struct blk_mq_ctx {
	struct {
		spinlock_t		lock;
		// 用于链接软件列表
		struct list_head	rq_lists[HCTX_MAX_TYPES];
	} ____cacheline_aligned_in_smp;

	// 该软件队列对应的cpu索引
	unsigned int		cpu;
	unsigned short		index_hw[HCTX_MAX_TYPES];
	// 指向HCTX_TYPE_DEFAULT、HCTX_TYPE_READ、HCTX_TYPE_POLL类型的硬件队列
	struct blk_mq_hw_ctx 	*hctxs[HCTX_MAX_TYPES];
	// 指向的request_queue
	struct request_queue	*queue;
	struct blk_mq_ctxs      *ctxs;
	struct kobject		kobj;
} ____cacheline_aligned_in_smp;
```

#### 3.1.4 struct blk_mq_hw_ctx
`struct blk_mq_hw_ctx`，用来表示硬件队列，每个硬件队列和`blk_mq_tags`一一对应
```c
/**
 * struct blk_mq_hw_ctx - State for a hardware queue facing the hardware
 * block device
 */
struct blk_mq_hw_ctx {
	struct {
		/** @lock: Protects the dispatch list. */
		spinlock_t		lock;
		/**
		 * @dispatch: Used for requests that are ready to be
		 * dispatched to the hardware but for some reason (e.g. lack of
		 * resources) could not be sent to the hardware. As soon as the
		 * driver can send new requests, requests at this list will
		 * be sent first for a fairer dispatch.
		 */
		// 硬件队列中request列表的头节点
		struct list_head	dispatch;
		 /**
		  * @state: BLK_MQ_S_* flags. Defines the state of the hw
		  * queue (active, scheduled to restart, stopped).
		  */
		unsigned long		state;
	} ____cacheline_aligned_in_smp;

	/**
	 * @run_work: Used for scheduling a hardware queue run at a later time.
	 */
	struct delayed_work	run_work;
	/** @cpumask: Map of available CPUs where this hctx can run. */
	cpumask_var_t		cpumask;
	/**
	 * @next_cpu: Used by blk_mq_hctx_next_cpu() for round-robin CPU
	 * selection from @cpumask.
	 */
	int			next_cpu;
	/**
	 * @next_cpu_batch: Counter of how many works left in the batch before
	 * changing to the next CPU.
	 */
	int			next_cpu_batch;

	/** @flags: BLK_MQ_F_* flags. Defines the behaviour of the queue. */
	unsigned long		flags;

	/**
	 * @sched_data: Pointer owned by the IO scheduler attached to a request
	 * queue. It's up to the IO scheduler how to use this pointer.
	 */
	void			*sched_data;
	/**
	 * @queue: Pointer to the request queue that owns this hardware context.
	 */
	struct request_queue	*queue;
	/** @fq: Queue of requests that need to perform a flush operation. */
	struct blk_flush_queue	*fq;

	/**
	 * @driver_data: Pointer to data owned by the block driver that created
	 * this hctx
	 */
	void			*driver_data;

	/**
	 * @ctx_map: Bitmap for each software queue. If bit is on, there is a
	 * pending request in that software queue.
	 */
	struct sbitmap		ctx_map;

	/**
	 * @dispatch_from: Software queue to be used when no scheduler was
	 * selected.
	 */
	struct blk_mq_ctx	*dispatch_from;
	/**
	 * @dispatch_busy: Number used by blk_mq_update_dispatch_busy() to
	 * decide if the hw_queue is busy using Exponential Weighted Moving
	 * Average algorithm.
	 */
	unsigned int		dispatch_busy;

	/** @type: HCTX_TYPE_* flags. Type of hardware queue. */
	unsigned short		type;
	/** @nr_ctx: Number of software queues. */
	unsigned short		nr_ctx;
	/** @ctxs: Array of software queues. */
	struct blk_mq_ctx	**ctxs;

	/** @dispatch_wait_lock: Lock for dispatch_wait queue. */
	spinlock_t		dispatch_wait_lock;
	/**
	 * @dispatch_wait: Waitqueue to put requests when there is no tag
	 * available at the moment, to wait for another try in the future.
	 */
	wait_queue_entry_t	dispatch_wait;

	/**
	 * @wait_index: Index of next available dispatch_wait queue to insert
	 * requests.
	 */
	atomic_t		wait_index;

	/**
	 * @tags: Tags owned by the block driver. A tag at this set is only
	 * assigned when a request is dispatched from a hardware queue.
	 */
	// 指向硬件队列对应的blk_mq_tags，针对无IO调度算法
	struct blk_mq_tags	*tags;
	/**
	 * @sched_tags: Tags owned by I/O scheduler. If there is an I/O
	 * scheduler associated with a request queue, a tag is assigned when
	 * that request is allocated. Else, this member is not used.
	 */
	//
	struct blk_mq_tags	*sched_tags;

	/** @run: Number of dispatched requests. */
	unsigned long		run;

	/** @numa_node: NUMA node the storage adapter has been connected to. */\
	// numa节点
	unsigned int		numa_node;
	/** @queue_num: Index of this hardware queue. */
	// 硬件队列索引号
	unsigned int		queue_num;

	/**
	 * @nr_active: Number of active requests. Only used when a tag set is
	 * shared across request queues.
	 */
	atomic_t		nr_active;

	/** @cpuhp_online: List to store request if CPU is going to die */
	struct hlist_node	cpuhp_online;
	/** @cpuhp_dead: List to store request if some CPU die. */
	struct hlist_node	cpuhp_dead;
	/** @kobj: Kernel object for sysfs. */
	struct kobject		kobj;

#ifdef CONFIG_BLK_DEBUG_FS
	/**
	 * @debugfs_dir: debugfs directory for this hardware queue. Named
	 * as cpu<cpu_number>.
	 */
	struct dentry		*debugfs_dir;
	/** @sched_debugfs_dir:	debugfs directory for the scheduler. */
	struct dentry		*sched_debugfs_dir;
#endif

	/**
	 * @hctx_list: if this hctx is not in use, this is an entry in
	 * q->unused_hctx_list.
	 */
	struct list_head	hctx_list;
};
```

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

## 块设备读写流程
![](https://raw.githubusercontent.com/JackHuang021/images/master/IO+request流程.png)

## 4 mq-deadline 调度算法

**`mq-deadline`调度器的特点：**
1. 对request划分了三个优先级（实时、普通、空闲）
2. 每个优先级将IO请求分为了read和write两种类型，对于每种类型的I/O分别有一颗红黑树和一个fifo队列，红黑树根据IO请求的起始扇区号进行排序，fifo队列记录了request进入调度器的顺序
3. read类型的request可以抢占write类型的request，但是会有一个饥饿计数防止write请求被“饿死”
4. 针对穿透性IO这种需要尽快发送到设备的IO设置另外一个dispatch队列，然后每次派发的时候都优先派发dispatch队列上的IO

mq-deadline适合于对实时性敏感的程序，它能保证IO在规定的时间内能下发下去

### 4.1 源码分析
基于linux 6.6，源码位置: `block/mq-deadline.c`

#### 4.1.1 数据结构
`struct deadline_data`，一个块设备对应一个`struct deadline_data`，存储了调度器的状态、请求队列等信息。
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
// block/elevator.h
/*
 * identifies an elevator type, such as AS or deadline
 */
struct elevator_type
{
	/* managed by elevator core */
	struct kmem_cache *icq_cache;

	/* fields provided by elevator implementation */
	struct elevator_mq_ops ops;

	size_t icq_size;	/* see iocontext.h */
	size_t icq_align;	/* ditto */
	struct elv_fs_entry *elevator_attrs;
	const char *elevator_name;
	const char *elevator_alias;
	const unsigned int elevator_features;
	struct module *elevator_owner;
#ifdef CONFIG_BLK_DEBUG_FS
	const struct blk_mq_debugfs_attr *queue_debugfs_attrs;
	const struct blk_mq_debugfs_attr *hctx_debugfs_attrs;
#endif

	/* managed by elevator core */
	char icq_cache_name[ELV_NAME_MAX + 6];	/* elvname + "_io_cq" */
	struct list_head list;
};

struct elevator_mq_ops {
	int (*init_sched)(struct request_queue *, struct elevator_type *);
	void (*exit_sched)(struct elevator_queue *);
	int (*init_hctx)(struct blk_mq_hw_ctx *, unsigned int);
	void (*exit_hctx)(struct blk_mq_hw_ctx *, unsigned int);
	void (*depth_updated)(struct blk_mq_hw_ctx *);

	bool (*allow_merge)(struct request_queue *, struct request *, struct bio *);
	bool (*bio_merge)(struct request_queue *, struct bio *, unsigned int);
	int (*request_merge)(struct request_queue *q, struct request **, struct bio *);
	void (*request_merged)(struct request_queue *, struct request *, enum elv_merge);
	void (*requests_merged)(struct request_queue *, struct request *, struct request *);
	void (*limit_depth)(blk_opf_t, struct blk_mq_alloc_data *);
	void (*prepare_request)(struct request *);
	void (*finish_request)(struct request *);
	void (*insert_requests)(struct blk_mq_hw_ctx *hctx, struct list_head *list,
			blk_insert_t flags);
	struct request *(*dispatch_request)(struct blk_mq_hw_ctx *);
	bool (*has_work)(struct blk_mq_hw_ctx *);
	void (*completed_request)(struct request *, u64);
	void (*requeue_request)(struct request *);
	struct request *(*former_request)(struct request_queue *, struct request *);
	struct request *(*next_request)(struct request_queue *, struct request *);
	void (*init_icq)(struct io_cq *);
	void (*exit_icq)(struct io_cq *);
};

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

#### 4.1.2 初始化mq-deadline
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

#### 4.1.3 bio合并到mq-deadline
到达block层的bio会首先尝试与现有的request进行合并，如果合并不成功再生成request，`bio_merge()`之后会依次调用`request_merge()`和`request_merged()`
```c
/*
 * Attempt to merge a bio into an existing request. This function is called
 * before @bio is associated with a request.
 */
static bool dd_bio_merge(struct request_queue *q, struct bio *bio,
		unsigned int nr_segs)
{
	struct deadline_data *dd = q->elevator->elevator_data;
	struct request *free = NULL;
	bool ret;

	spin_lock(&dd->lock);
	// 调用通用处理流程的合并函数
	ret = blk_mq_sched_try_merge(q, bio, nr_segs, &free);
	spin_unlock(&dd->lock);

	if (free)
		blk_mq_free_request(free);

	return ret;
}

bool blk_mq_sched_try_merge(struct request_queue *q, struct bio *bio,
		unsigned int nr_segs, struct request **merged_request)
{
	struct request *rq;

	switch (elv_merge(q, &rq, bio)) {
	case ELEVATOR_BACK_MERGE:
		if (!blk_mq_sched_allow_merge(q, rq, bio))
			return false;
		if (bio_attempt_back_merge(rq, bio, nr_segs) != BIO_MERGE_OK)
			return false;
		*merged_request = attempt_back_merge(q, rq);
		// request_merged当request成功地与bio或者某个相邻的request合并之后调用
		if (!*merged_request)
			elv_merged_request(q, rq, ELEVATOR_BACK_MERGE);
		return true;
	case ELEVATOR_FRONT_MERGE:
		if (!blk_mq_sched_allow_merge(q, rq, bio))
			return false;
		if (bio_attempt_front_merge(rq, bio, nr_segs) != BIO_MERGE_OK)
			return false;
		*merged_request = attempt_front_merge(q, rq);
		if (!*merged_request)
			elv_merged_request(q, rq, ELEVATOR_FRONT_MERGE);
		return true;
	case ELEVATOR_DISCARD_MERGE:
		return bio_attempt_discard_merge(q, rq, bio) == BIO_MERGE_OK;
	default:
		return false;
	}
}

// 调用到elv_merge()来判断是否可以合并，并返回结果
enum elv_merge elv_merge(struct request_queue *q, struct request **req,
		struct bio *bio)
{
	struct elevator_queue *e = q->elevator;
	struct request *__rq;

	/*
	 * Levels of merges:
	 * 	nomerges:  No merges at all attempted
	 * 	noxmerges: Only simple one-hit cache try
	 * 	merges:	   All merge tries attempted
	 */
	if (blk_queue_nomerges(q) || !bio_mergeable(bio))
		return ELEVATOR_NO_MERGE;

	/*
	 * First try one-hit cache.
	 */
	// 先判断上次合并的request是否符合合并的要求
	if (q->last_merge && elv_bio_merge_ok(q->last_merge, bio)) {
		// bio的块设备起始扇区必须和request上处理的sector相邻才能被合并
		enum elv_merge ret = blk_try_merge(q->last_merge, bio);

		if (ret != ELEVATOR_NO_MERGE) {
			*req = q->last_merge;
			return ret;
		}
	}

	if (blk_queue_noxmerges(q))
		return ELEVATOR_NO_MERGE;

	/*
	 * See if our hash lookup can find a potential backmerge.
	 */
	// 如果last_merge不符合要求再从其余request中找
	__rq = elv_rqhash_find(q, bio->bi_iter.bi_sector);
	if (__rq && elv_bio_merge_ok(__rq, bio)) {
		*req = __rq;

		if (blk_discard_mergable(__rq))
			return ELEVATOR_DISCARD_MERGE;
		return ELEVATOR_BACK_MERGE;
	}
	// 调用request_merge()接口，这里是找可以前向合并的request
	if (e->type->ops.request_merge)
		return e->type->ops.request_merge(q, req, bio);

	return ELEVATOR_NO_MERGE;
}

/*
 * Try to merge @bio into an existing request. If @bio has been merged into
 * an existing request, store the pointer to that request into *@rq.
 */
static int dd_request_merge(struct request_queue *q, struct request **rq,
			    struct bio *bio)
{
	struct deadline_data *dd = q->elevator->elevator_data;
	const u8 ioprio_class = IOPRIO_PRIO_CLASS(bio->bi_ioprio);
	const enum dd_prio prio = ioprio_class_to_prio[ioprio_class];
	struct dd_per_prio *per_prio = &dd->per_prio[prio];
	sector_t sector = bio_end_sector(bio);
	struct request *__rq;

	if (!dd->front_merges)
		return ELEVATOR_NO_MERGE;
	//从红黑树中找到一个起始扇区是bio的结束扇区的request，表明bio可以front merge到request
	__rq = elv_rb_find(&per_prio->sort_list[bio_data_dir(bio)], sector);
	if (__rq) {
		BUG_ON(sector != blk_rq_pos(__rq));

		if (elv_bio_merge_ok(__rq, bio)) {
			*rq = __rq;
			if (blk_discard_mergable(__rq))
				return ELEVATOR_DISCARD_MERGE;
			return ELEVATOR_FRONT_MERGE;
		}
	}

	return ELEVATOR_NO_MERGE;
}
```


#### 4.1.4 request插入到mq-deadline
当bio不能合并到现有的request时，会生成一个新的request，然后将request插入到mq-deadline中等待调度
```c
// block/mq-deadline.c
/*
 * Called from blk_mq_insert_request() or blk_mq_dispatch_plug_list().
 */
static void dd_insert_requests(struct blk_mq_hw_ctx *hctx,
			       struct list_head *list,
			       blk_insert_t flags)
{
	struct request_queue *q = hctx->queue;
	struct deadline_data *dd = q->elevator->elevator_data;
	LIST_HEAD(free);

	spin_lock(&dd->lock);
	while (!list_empty(list)) {
		struct request *rq;

		rq = list_first_entry(list, struct request, queuelist);
		list_del_init(&rq->queuelist);
		dd_insert_request(hctx, rq, flags, &free);
	}
	spin_unlock(&dd->lock);

	blk_mq_free_requests(&free);
}

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
		// 普通io插入到mq-deadline的红黑树
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

#### 4.1.5 mq-deadline分发request
硬件队列派发后会调用调度器的dispatch_request接口获得一个可以派发的request
```c
// block/mq-deadline.c
/*
 * Called from blk_mq_run_hw_queue() -> __blk_mq_sched_dispatch_requests().
 *
 * One confusing aspect here is that we get called for a specific
 * hardware queue, but we may return a request that is for a
 * different hardware queue. This is because mq-deadline has shared
 * state for all hardware queues, in terms of sorting, FIFOs, etc.
 */
static struct request *dd_dispatch_request(struct blk_mq_hw_ctx *hctx)
{
	struct deadline_data *dd = hctx->queue->elevator->elevator_data;
	const unsigned long now = jiffies;
	struct request *rq;
	enum dd_prio prio;

	spin_lock(&dd->lock);
	rq = dd_dispatch_prio_aged_requests(dd, now);
	if (rq)
		goto unlock;

	/*
	 * Next, dispatch requests in priority order. Ignore lower priority
	 * requests if any higher priority requests are pending.
	 */
	// 按照优先级进行分发，低优先级的先等待
	for (prio = 0; prio <= DD_PRIO_MAX; prio++) {
		rq = __dd_dispatch_request(dd, &dd->per_prio[prio], now);
		if (rq || dd_queued(dd, prio))
			break;
	}

unlock:
	spin_unlock(&dd->lock);

	return rq;
}

/*
 * deadline_dispatch_requests selects the best request according to
 * read/write expire, fifo_batch, etc and with a start time <= @latest_start.
 */
static struct request *__dd_dispatch_request(struct deadline_data *dd,
					     struct dd_per_prio *per_prio,
					     unsigned long latest_start)
{
	struct request *rq, *next_rq;
	enum dd_data_dir data_dir;
	enum dd_prio prio;
	u8 ioprio_class;

	lockdep_assert_held(&dd->lock);
	// 如果dispatch队列不为空，则优先派发dispatch队列上的穿透性IO
	if (!list_empty(&per_prio->dispatch)) {
		rq = list_first_entry(&per_prio->dispatch, struct request,
				      queuelist);
		if (started_after(dd, rq, latest_start))
			return NULL;
		list_del_init(&rq->queuelist);
		data_dir = rq_data_dir(rq);
		goto done;
	}

	/*
	 * batches are currently reads XOR writes
	 */
	// 按照request的扇区顺序依次派发
	rq = deadline_next_request(dd, per_prio, dd->last_dir);
	// 如果批量派发的个数在规定限制内，则可以派发
	if (rq && dd->batching < dd->fifo_batch) {
		/* we have a next request and are still entitled to batch */
		data_dir = rq_data_dir(rq);
		goto dispatch_request;
	}

	/*
	 * at this point we are not running a batch. select the appropriate
	 * data direction (read / write)
	 */
	// 优先看read队列是否由到期的IO
	if (!list_empty(&per_prio->fifo_list[DD_READ])) {
		BUG_ON(RB_EMPTY_ROOT(&per_prio->sort_list[DD_READ]));
		// 如果write队列有到期的IO，并且read让write“饥饿”的次数超过了2次，则去派发write
		if (deadline_fifo_request(dd, per_prio, DD_WRITE) &&
		    (dd->starved++ >= dd->writes_starved))
			goto dispatch_writes;

		data_dir = DD_READ;

		goto dispatch_find_request;
	}

	/*
	 * there are either no reads or writes have been starved
	 */
	// 派发write，并将starved置为0
	if (!list_empty(&per_prio->fifo_list[DD_WRITE])) {
dispatch_writes:
		BUG_ON(RB_EMPTY_ROOT(&per_prio->sort_list[DD_WRITE]));

		dd->starved = 0;

		data_dir = DD_WRITE;

		goto dispatch_find_request;
	}

	return NULL;

dispatch_find_request:
	/*
	 * we are not running a batch, find best request for selected data_dir
	 */
	next_rq = deadline_next_request(dd, per_prio, data_dir);
	 // 如果队列有到期的IO，或者批量派发没有下一个IO了则从fifo队列里取出第一个IO来派发
	if (deadline_check_fifo(per_prio, data_dir) || !next_rq) {
		/*
		 * A deadline has expired, the last request was in the other
		 * direction, or we have run out of higher-sectored requests.
		 * Start again from the request with the earliest expiry time.
		 */
		rq = deadline_fifo_request(dd, per_prio, data_dir);
	} else {
		/*
		 * The last req was the same dir and we have a next request in
		 * sort order. No expired requests so continue on from here.
		 */
		rq = next_rq;
	}

	/*
	 * For a zoned block device, if we only have writes queued and none of
	 * them can be dispatched, rq will be NULL.
	 */
	if (!rq)
		return NULL;

	dd->last_dir = data_dir;
	dd->batching = 0;

dispatch_request:
	if (started_after(dd, rq, latest_start))
		return NULL;

	/*
	 * rq is the selected appropriate request.
	 */
	dd->batching++;
	deadline_move_request(dd, per_prio, rq);
done:
	ioprio_class = dd_rq_ioclass(rq);
	prio = ioprio_class_to_prio[ioprio_class];
	dd->per_prio[prio].latest_pos[data_dir] = blk_rq_pos(rq);
	dd->per_prio[prio].stats.dispatched++;
	/*
	 * If the request needs its target zone locked, do it.
	 */
	blk_req_zone_write_lock(rq);
	rq->rq_flags |= RQF_STARTED;
	return rq;
}
```

## 5. kyber 调度算法
kyber调度器创建read、write、discard、other四个队列将IO分类处理。kyber不是消耗光了某个队列再去分发下一个队列，而是消耗到一定的个数就切换到下一个队列，从而防止后面的队列被饿死，这个个数分别是16、8、1、1，也就是分发了16个读IO之后去分发写，分发了8个写之后再分发一个discard，最后分发一个other的IO，以此类推循环。

### 5.1 源码分析

`struct kyber_queue_data`，kyber的主要数据结构
```c
// block/kyber-iosched.c

// 四种类型的IO
static const char *kyber_domain_names[] = {
	[KYBER_READ] = "READ",
	[KYBER_WRITE] = "WRITE",
	[KYBER_DISCARD] = "DISCARD",
	[KYBER_OTHER] = "OTHER",
};

struct kyber_queue_data {
	// 指向块设备对应的request_queue
	struct request_queue *q;
	dev_t dev;

	/*
	 * Each scheduling domain has a limited number of in-flight requests
	 * device-wide, limited by these tokens.
	 */
	// 每种队列的token占用情况，分发IO时从这里申请
	struct sbitmap_queue domain_tokens[KYBER_NUM_DOMAINS];

	/*
	 * Async request percentage, converted to per-word depth for
	 * sbitmap_get_shallow().
	 */
	// 用于限制异步请求的带宽，防止同步请求被饿死
	unsigned int async_depth;
	// 统计时延信息，IO完成时就会统计时延保存到这里
	struct kyber_cpu_latency __percpu *cpu_latency;

	/* Timer for stats aggregation and adjusting domain tokens. */
	// 使用定时器每隔一段时间统计一下时延情况，根据统计情况调整token数量
	struct timer_list timer;

	unsigned int latency_buckets[KYBER_OTHER][2][KYBER_LATENCY_BUCKETS];
	// 记录上一次调整token的时间
	unsigned long latency_timeout[KYBER_OTHER];
	// 记录上一次timer得到的时延好坏结果
	int domain_p99[KYBER_OTHER];

	/* Target latencies in nanoseconds. */
	// 每种IO类型的时延参考值
	u64 latency_targets[KYBER_OTHER];
};
```

`struct kyber_hctx_data`存放于硬件队列`struct blk_mq_hw_ctx`的sched_data字段，包含了暂存队列和分发队列
```c
// block/kyber-iosched.c
struct kyber_hctx_data {
	spinlock_t lock;
	// 分发队列，IO从这个队列提交到硬件队列
	struct list_head rqs[KYBER_NUM_DOMAINS];
	// 当前分发的IO类型
	unsigned int cur_domain;
	// 记录当前暂存队列已经派发的IO个数
	unsigned int batching;
	// 暂存队列，硬队列对应的软队列有多少个就有多少个暂存队列
	struct kyber_ctx_queue *kcqs;
	// 用于表示暂存队列上是否有IO
	struct sbitmap kcq_map[KYBER_NUM_DOMAINS];
	struct sbq_wait domain_wait[KYBER_NUM_DOMAINS];
	struct sbq_wait_state *domain_ws[KYBER_NUM_DOMAINS];
	atomic_t wait_index[KYBER_NUM_DOMAINS];
};

// 暂存队列数据结构
/*
 * There is a same mapping between ctx & hctx and kcq & khd,
 * we use request->mq_ctx->index_hw to index the kcq in khd.
 */
struct kyber_ctx_queue {
	/*
	 * Used to ensure operations on rq_list and kcq_map to be an atmoic one.
	 * Also protect the rqs on rq_list when merge.
	 */
	spinlock_t lock;
	struct list_head rq_list[KYBER_NUM_DOMAINS];
} ____cacheline_aligned_in_smp;

```

kyber IO调度算法实例
```c
// block/kyber-iosched.c
static struct elevator_type kyber_sched = {
	.ops = {
		.init_sched = kyber_init_sched,
		.exit_sched = kyber_exit_sched,
		.init_hctx = kyber_init_hctx,
		.exit_hctx = kyber_exit_hctx,
		.limit_depth = kyber_limit_depth,
		.bio_merge = kyber_bio_merge,
		.prepare_request = kyber_prepare_request,
		.insert_requests = kyber_insert_requests,
		.finish_request = kyber_finish_request,
		.requeue_request = kyber_finish_request,
		.completed_request = kyber_completed_request,
		.dispatch_request = kyber_dispatch_request,
		.has_work = kyber_has_work,
		.depth_updated = kyber_depth_updated,
	},
#ifdef CONFIG_BLK_DEBUG_FS
	.queue_debugfs_attrs = kyber_queue_debugfs_attrs,
	.hctx_debugfs_attrs = kyber_hctx_debugfs_attrs,
#endif
	.elevator_attrs = kyber_sched_attrs,
	.elevator_name = "kyber",
	.elevator_owner = THIS_MODULE,
};
```

#### 5.1.1 kyber初始化
当块设备的调度器被设置成kyber时会调用`kyber_init_sched()`函数初始化`kyber_queue_data`，将`kyber_queue_data`与`request_queue`绑定。
```c
// block/kyber-iosched.c
static int kyber_init_sched(struct request_queue *q, struct elevator_type *e)
{
	struct kyber_queue_data *kqd;
	struct elevator_queue *eq;

	eq = elevator_alloc(q, e);
	if (!eq)
		return -ENOMEM;

	kqd = kyber_queue_data_alloc(q);
	if (IS_ERR(kqd)) {
		kobject_put(&eq->kobj);
		return PTR_ERR(kqd);
	}

	blk_stat_enable_accounting(q);

	blk_queue_flag_clear(QUEUE_FLAG_SQ_SCHED, q);

	eq->elevator_data = kqd;
	q->elevator = eq;

	return 0;
}

static struct kyber_queue_data *kyber_queue_data_alloc(struct request_queue *q)
{
	struct kyber_queue_data *kqd;
	int ret = -ENOMEM;
	int i;

	kqd = kzalloc_node(sizeof(*kqd), GFP_KERNEL, q->node);
	if (!kqd)
		goto err;

	kqd->q = q;
	kqd->dev = disk_devt(q->disk);

	kqd->cpu_latency = alloc_percpu_gfp(struct kyber_cpu_latency,
					    GFP_KERNEL | __GFP_ZERO);
	if (!kqd->cpu_latency)
		goto err_kqd;

	timer_setup(&kqd->timer, kyber_timer_fn, 0);
	// 初始化每种队列的token数，kyber_depth全局变量显示为256、128、64、16
	for (i = 0; i < KYBER_NUM_DOMAINS; i++) {
		WARN_ON(!kyber_depth[i]);
		WARN_ON(!kyber_batch_size[i]);
		ret = sbitmap_queue_init_node(&kqd->domain_tokens[i],
					      kyber_depth[i], -1, false,
					      GFP_KERNEL, q->node);
		if (ret) {
			while (--i >= 0)
				sbitmap_queue_free(&kqd->domain_tokens[i]);
			goto err_buckets;
		}
	}
	// 初始化总的时延统计和每种队列的时延参考值
	for (i = 0; i < KYBER_OTHER; i++) {
		kqd->domain_p99[i] = -1;
		kqd->latency_targets[i] = kyber_latency_targets[i];
	}

	return kqd;

err_buckets:
	free_percpu(kqd->cpu_latency);
err_kqd:
	kfree(kqd);
err:
	return ERR_PTR(ret);
}
```

#### 5.1.2 初始化kyber_hctx_data
```c
// block/kyber-iosched.c
static int kyber_init_hctx(struct blk_mq_hw_ctx *hctx, unsigned int hctx_idx)
{
	struct kyber_hctx_data *khd;
	int i;

	khd = kmalloc_node(sizeof(*khd), GFP_KERNEL, hctx->numa_node);
	if (!khd)
		return -ENOMEM;

	khd->kcqs = kmalloc_array_node(hctx->nr_ctx,
				       sizeof(struct kyber_ctx_queue),
				       GFP_KERNEL, hctx->numa_node);
	if (!khd->kcqs)
		goto err_khd;
	// 初始化暂存队列
	for (i = 0; i < hctx->nr_ctx; i++)
		kyber_ctx_queue_init(&khd->kcqs[i]);
	// 初始化kcq_map，用来记录暂存队列上是否有IO挂着
	for (i = 0; i < KYBER_NUM_DOMAINS; i++) {
		if (sbitmap_init_node(&khd->kcq_map[i], hctx->nr_ctx,
				      ilog2(8), GFP_KERNEL, hctx->numa_node,
				      false, false)) {
			while (--i >= 0)
				sbitmap_free(&khd->kcq_map[i]);
			goto err_kcqs;
		}
	}

	spin_lock_init(&khd->lock);

	for (i = 0; i < KYBER_NUM_DOMAINS; i++) {
		INIT_LIST_HEAD(&khd->rqs[i]);
		khd->domain_wait[i].sbq = NULL;
		init_waitqueue_func_entry(&khd->domain_wait[i].wait,
					  kyber_domain_wake);
		khd->domain_wait[i].wait.private = hctx;
		INIT_LIST_HEAD(&khd->domain_wait[i].wait.entry);
		atomic_set(&khd->wait_index[i], 0);
	}

	khd->cur_domain = 0;
	khd->batching = 0;

	hctx->sched_data = khd;
	kyber_depth_updated(hctx);

	return 0;

err_kcqs:
	kfree(khd->kcqs);
err_khd:
	kfree(khd);
	return -ENOMEM;
}
```

#### 5.1.3 bio合入kyber
```c
static bool kyber_bio_merge(struct request_queue *q, struct bio *bio,
		unsigned int nr_segs)
{
	// 根据软队列在硬队列里的下标找到应该合并哪个暂存队列
	struct blk_mq_ctx *ctx = blk_mq_get_ctx(q);
	struct blk_mq_hw_ctx *hctx = blk_mq_map_queue(q, bio->bi_opf, ctx);
	struct kyber_hctx_data *khd = hctx->sched_data;
	struct kyber_ctx_queue *kcq = &khd->kcqs[ctx->index_hw[hctx->type]];
	// 根据请求操作标志找到是read、write、discard还是other
	unsigned int sched_domain = kyber_sched_domain(bio->bi_opf);
	struct list_head *rq_list = &kcq->rq_list[sched_domain];
	bool merged;

	spin_lock(&kcq->lock);
	// 调用block层通用函数去合并bio到某个request
	merged = blk_bio_list_merge(hctx->queue, rq_list, bio, nr_segs);
	spin_unlock(&kcq->lock);

	return merged;
}
```

#### 5.1.4 bio插入到kyber
```c
static void kyber_insert_requests(struct blk_mq_hw_ctx *hctx,
				  struct list_head *rq_list,
				  blk_insert_t flags)
{
	struct kyber_hctx_data *khd = hctx->sched_data;
	struct request *rq, *next;

	list_for_each_entry_safe(rq, next, rq_list, queuelist) {
		// 先找到sched_domain和kyber_ctx_queue
		unsigned int sched_domain = kyber_sched_domain(rq->cmd_flags);
		struct kyber_ctx_queue *kcq = &khd->kcqs[rq->mq_ctx->index_hw[hctx->type]];
		struct list_head *head = &kcq->rq_list[sched_domain];

		spin_lock(&kcq->lock);
		trace_block_rq_insert(rq);
		// 将request插入到队列上
		if (flags & BLK_MQ_INSERT_AT_HEAD)
			list_move(&rq->queuelist, head);
		else
			list_move_tail(&rq->queuelist, head);
		// 设置bit表示暂存队列上有request
		sbitmap_set_bit(&khd->kcq_map[sched_domain],
				rq->mq_ctx->index_hw[hctx->type]);
		spin_unlock(&kcq->lock);
	}
}
```

#### 5.1.5 kyber分发request
kyber采用负载均衡的方式遍历分发队列的read、write、discard、other队列，选择一个IO分发到硬队列，当分发队列上没有IO时会遍历与这个分发队列相关联的所有暂存队列，将暂存队列上的所有IO都转到分发队列上，然后再看有没有IO可以分发的。
```c
static struct request *kyber_dispatch_request(struct blk_mq_hw_ctx *hctx)
{
	struct kyber_queue_data *kqd = hctx->queue->elevator->elevator_data;
	struct kyber_hctx_data *khd = hctx->sched_data;
	struct request *rq;
	int i;

	spin_lock(&khd->lock);

	/*
	 * First, if we are still entitled to batch, try to dispatch a request
	 * from the batch.
	 */
	// 如果当前队列派发的IO个数还没有达到最大值则继续派发当前队列的IO
	if (khd->batching < kyber_batch_size[khd->cur_domain]) {
		rq = kyber_dispatch_cur_domain(kqd, khd, hctx);
		if (rq)
			goto out;
	}

	/*
	 * Either,
	 * 1. We were no longer entitled to a batch.
	 * 2. The domain we were batching didn't have any requests.
	 * 3. The domain we were batching was out of tokens.
	 *
	 * Start another batch. Note that this wraps back around to the original
	 * domain if no other domains have requests or tokens.
	 */
	// 已经达到派发最大值，将batching置0，继续派发下一个暂存队列
	khd->batching = 0;
	for (i = 0; i < KYBER_NUM_DOMAINS; i++) {
		if (khd->cur_domain == KYBER_NUM_DOMAINS - 1)
			khd->cur_domain = 0;
		else
			khd->cur_domain++;

		rq = kyber_dispatch_cur_domain(kqd, khd, hctx);
		if (rq)
			goto out;
	}

	rq = NULL;
out:
	spin_unlock(&khd->lock);
	return rq;
}

static struct request *
kyber_dispatch_cur_domain(struct kyber_queue_data *kqd,
			  struct kyber_hctx_data *khd,
			  struct blk_mq_hw_ctx *hctx)
{
	struct list_head *rqs;
	struct request *rq;
	int nr;
	// 获取当前的派发队列
	rqs = &khd->rqs[khd->cur_domain];

	/*
	 * If we already have a flushed request, then we just need to get a
	 * token for it. Otherwise, if there are pending requests in the kcqs,
	 * flush the kcqs, but only if we can get a token. If not, we should
	 * leave the requests in the kcqs so that they can be merged. Note that
	 * khd->lock serializes the flushes, so if we observed any bit set in
	 * the kcq_map, we will always get a request.
	 */
	// 遍历request进行派发
	rq = list_first_entry_or_null(rqs, struct request, queuelist);
	if (rq) {
		// 获取token
		nr = kyber_get_domain_token(kqd, khd, hctx);
		if (nr >= 0) {
			khd->batching++;
			// 将token保存在request的priv字段里面
			rq_set_domain_token(rq, nr);
			list_del_init(&rq->queuelist);
			return rq;
		} else {
			trace_kyber_throttled(kqd->dev,
					      kyber_domain_names[khd->cur_domain]);
		}
	} else if (sbitmap_any_bit_set(&khd->kcq_map[khd->cur_domain])) {
		nr = kyber_get_domain_token(kqd, khd, hctx);
		if (nr >= 0) {
			// 暂存队列有IO，并且当前IO类型的token还没有被消耗完
            // 将暂存队列的IO转到分发队列上
			kyber_flush_busy_kcqs(khd, khd->cur_domain, rqs);
			rq = list_first_entry(rqs, struct request, queuelist);
			khd->batching++;
			rq_set_domain_token(rq, nr);
			list_del_init(&rq->queuelist);
			return rq;
		} else {
			trace_kyber_throttled(kqd->dev,
					      kyber_domain_names[khd->cur_domain]);
		}
	}

	/* There were either no pending requests or no tokens. */
	return NULL;
}
```

## 6. bfq（Budget Fair Queueing）调度算法
bfq全称Budget Fair Queueing，是Paolo Valente在2010年提出的一个IO调度器，目标是取代cfq。这里的“Budget”是指磁盘扇区sector，“Budget Fair”是指存储设备公平地对待每个进程，为各个进程服务相同数量的sector。

bfq的基本原理是：每个进程都先分配一个bfq调度队列bfq_queue，简称bfqq，bfqq与进程绑定。每个进程的bfqq分配一个初始配额budget，进程每派发一个IO请求，就消耗bfqq的一定配额(消耗的配额与传输的IO请求数据量成正比)。等bfqq的配额消耗光、bfqq上没有IO请求要要传输，则令bfqq到期失效。接着切换到其他进程的bfqq，派发这个新的bfqq上的IO请求。“Budget Fair”是指存储设备公平地对待每个进程，为各个进程服务相同数量的sector。

BFQ逻辑图
![](https://raw.githubusercontent.com/JackHuang021/images/master/bfq.png)

1. 实线箭头代表IO请求的路径方向。这与其他的调度器一样，都是通过add_request类函数，将IO请求插入到进程的IO队列中，然后通过调度器的调度，由 dispatch函数下发到驱动处理。
2. 虚线箭头代表bfq特有的budget特性。每个进程被分配了一定数量的budget，当该进程被bfq选择执行io时，最多只能访问这么多个budget。没访问一个sector，budget减1，budget用完了，bfq就会选择其他的进程执行io。执行io请求的进程由于某种原因过期时（budget用完了是一种原因），会基于上一次用了多少个budget重新估算下一次的budget数量。

调度器的设计离不开两个核心指标：高吞吐量、快速响应，但这两个指标是矛盾的，如何平衡这两个指标成了调度器首先要考虑的事。bfq通过Budget based Worst-case Weighted fair Queueing (b-wf2q+)算法尽可能地最优化这两个指标，自动识别出batch类进程、交互式进程，确保交互式进程快速响应的前提下，尽可能地保证batch类进程的高吞吐量。


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

## 8. 参考链接
1. [linux块设备驱动blk-mq](https://www.cnblogs.com/zyly/p/16690841.html#_label0_0)
2. [mq-deadline调度器原理及源码分析](https://www.cnblogs.com/kanie/p/15252921.html)
3. [kyber调度器原理及源码分析](https://www.cnblogs.com/kanie/p/15232058.html)
4. [https://www.cnblogs.com/Linux-tech/p/12961283.html](https://www.cnblogs.com/Linux-tech/p/12961283.html)
5. [IO子系统全流程介绍](https://zhuanlan.zhihu.com/p/545906763)