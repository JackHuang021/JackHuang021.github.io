---
title: linux ida和xarray实现
date: 2023-09-14 14:03:48
tags:
  - Linux
  - idr
categories: Linux
---

## 1. xarray
### 1.1 xarray介绍
xarray是一种高效的键值对数据结构，用于解决大规模数据集上的高并发访问问题，被广泛应用在linux内核的各个子系统，如文件系统、网络子系统和内存管理

### 1.2 xarray数据结构
xarray主要由`strutc xarray`和`struct xa_node`这两个结构体来实现

`struct xa_node`，负责存储数据或索引下一层的节点
```c
// include/linux/xarray.h
/*
 * @count is the count of every non-NULL element in the ->slots array
 * whether that is a value entry, a retry entry, a user pointer,
 * a sibling entry or a pointer to the next level of the tree.
 * @nr_values is the count of every element in ->slots which is
 * either a value entry or a sibling of a value entry.
 */
struct xa_node {
	unsigned char	shift;		/* Bits remaining in each slot */
	unsigned char	offset;		/* Slot offset in parent */
	unsigned char	count;		/* Total entry count */
	unsigned char	nr_values;	/* Value entry count */
	struct xa_node __rcu *parent;	/* NULL at top of tree */
	struct xarray	*array;		/* The array we belong to */
	union {
		struct list_head private_list;	/* For tree user */
		struct rcu_head	rcu_head;	/* Used when freeing node */
	};
	void __rcu	*slots[XA_CHUNK_SIZE];
	union {
		unsigned long	tags[XA_MAX_MARKS][XA_MARK_LONGS];
		unsigned long	marks[XA_MAX_MARKS][XA_MARK_LONGS];
	};
};
```

+ shift: 指定当前xa_node的slots数组中成员的单位，当shift为0时，说明当前slots数组中成员为叶子节点，当shift为6时，说明当前slots数组中成员指向的xa_node可以最多包含2^6个节点
+ offset: xa_node在父节点slots数组中的偏移
+ count: xa_node有多少个slots已经被使用
+ nr_values: xa_node有多少个slots存储的value entry
+ parent: 只想该xa_node的父节点
+ array: array成员指向该xa_node所属的xarray
+ slots: slots是一个指针数组，该数组可以存储下一级的节点，也可以用于存储即将插入的对象指针


`struct xarray`，整个数据结构的根，负责管理整个树形结构
```c
/**
 * struct xarray - The anchor of the XArray.
 * @xa_lock: Lock that protects the contents of the XArray.
 *
 * To use the xarray, define it statically or embed it in your data structure.
 * It is a very small data structure, so it does not usually make sense to
 * allocate it separately and keep a pointer to it in your data structure.
 *
 * You may use the xa_lock to protect your own data structures as well.
 */
/*
 * If all of the entries in the array are NULL, @xa_head is a NULL pointer.
 * If the only non-NULL entry in the array is at index 0, @xa_head is that
 * entry.  If any other entry in the array is non-NULL, @xa_head points
 * to an @xa_node.
 */
struct xarray {
	// 用于同步写操作，防止多核情况下的数据竞争
	spinlock_t	xa_lock;
/* private: The rest of the data structure is not to be used directly. */
	gfp_t		xa_flags;
	// 树的根节点，可能指向xa_node节点或者直接存储数据
	void __rcu *	xa_head;
};

// xa_flags的定义
/*
 * Values for xa_flags.  The radix tree stores its GFP flags in the xa_flags,
 * and we remain compatible with that.
 */
// 该标志表示操作时需要禁用本地中断
#define XA_FLAGS_LOCK_IRQ	((__force gfp_t)XA_LOCK_IRQ)
#define XA_FLAGS_LOCK_BH	((__force gfp_t)XA_LOCK_BH)
// 该标志用于追踪 XArray 中的空闲条目。
// 启用这个标志后，XArray 会记录哪些条目被释放，允许调用者更方便地找到和管理空闲的索引
#define XA_FLAGS_TRACK_FREE	((__force gfp_t)4U)
#define XA_FLAGS_ZERO_BUSY	((__force gfp_t)8U)
#define XA_FLAGS_ALLOC_WRAPPED	((__force gfp_t)16U)
#define XA_FLAGS_ACCOUNT	((__force gfp_t)32U)
#define XA_FLAGS_MARK(mark)	((__force gfp_t)((1U << __GFP_BITS_SHIFT) << \
						(__force unsigned)(mark)))
```

`struct xa_state`，记录了xarray操作过程中的状态
```c
/*
 * The xa_state is opaque to its users.  It contains various different pieces
 * of state involved in the current operation on the XArray.  It should be
 * declared on the stack and passed between the various internal routines.
 * The various elements in it should not be accessed directly, but only
 * through the provided accessor functions.  The below documentation is for
 * the benefit of those working on the code, not for users of the XArray.
 *
 * @xa_node usually points to the xa_node containing the slot we're operating
 * on (and @xa_offset is the offset in the slots array).  If there is a
 * single entry in the array at index 0, there are no allocated xa_nodes to
 * point to, and so we store %NULL in @xa_node.  @xa_node is set to
 * the value %XAS_RESTART if the xa_state is not walked to the correct
 * position in the tree of nodes for this operation.  If an error occurs
 * during an operation, it is set to an %XAS_ERROR value.  If we run off the
 * end of the allocated nodes, it is set to %XAS_BOUNDS.
 */
struct xa_state {
	// 指向xarray
	struct xarray *xa;
	// 存储当前待操作数据的索引
	unsigned long xa_index;
	unsigned char xa_shift;
	unsigned char xa_sibs;
	unsigned char xa_offset;
	unsigned char xa_pad;		/* Helps gcc generate better code */
	struct xa_node *xa_node;
	// 保存当前需要创建的xa_node指针
	struct xa_node *xa_alloc;
	xa_update_node_t xa_update;
	struct list_lru *xa_lru;
};
```

`xa_state`的创建宏`XA_STATE(name, array, index)`
```c
// include/linux/xarray.h

/**
 * XA_STATE() - Declare an XArray operation state.
 * @name: Name of this operation state (usually xas).
 * @array: Array to operate on.
 * @index: Initial index of interest.
 *
 * Declare and initialise an xa_state on the stack.
 */
#define XA_STATE(name, array, index)				\
	struct xa_state name = __XA_STATE(array, index, 0, 0)

#define __XA_STATE(array, index, shift, sibs)  {	\
	.xa = array,					\
	.xa_index = index,				\
	.xa_shift = shift,				\
	.xa_sibs = sibs,				\
	.xa_offset = 0,					\
	.xa_pad = 0,					\
	.xa_node = XAS_RESTART,				\
	.xa_alloc = NULL,				\
	.xa_update = NULL,				\
	.xa_lru = NULL,					\
}
```

### 1.3 xarray中的一些特殊条目
在xarray中，引入了一些特殊条目，这些条目由实际的指针值和一些特殊的标志位结合使用，表示特定的含义，这种设计方式通过位操作来节省内存并加快查找速度，同时还能提供额外的状态或者操作信息。

根据值（entry）的最后两位来对entry进行分类
+ 00: Pointer
+ x1: Value
+ 10: Internal

对于internal的entry将其右移2位后得到的值还会进行下面的判断
+ 0-255: sibling entry, advanced entry
+ 256: retry entry
+ 257: zero entry
+ >4096: node entry
+ >= -MAX_ERRNO: 不会存放在xa_node->slots中



`xa_is_internal()`，判断是否为内部条目，判断的依据是，第2位为1表示为一个内部条目
```c
// include/linux/xarray.h

/*
 * xa_is_internal() - Is the entry an internal entry?
 * @entry: XArray entry.
 *
 * Context: Any context.
 * Return: %true if the entry is an internal entry.
 */
static inline bool xa_is_internal(const void *entry)
{
	return ((unsigned long)entry & 3) == 2;
}
```

`xa_is_advanced()`，判断是否为高级条目，而不是用户存储的普通数据条目。高级条目的定义：必须要为内部条目，且其值小于`XA_RETRY_ENTRY`即1026
```c
// include/linux/xarray.h

/*
 * xa_mk_internal() - Create an internal entry.
 * @v: Value to turn into an internal entry.
 *
 * Internal entries are used for a number of purposes.  Entries 0-255 are
 * used for sibling entries (only 0-62 are used by the current code).  256
 * is used for the retry entry.  257 is used for the reserved / zero entry.
 * Negative internal entries are used to represent errnos.  Node pointers
 * are also tagged as internal entries in some situations.
 *
 * Context: Any context.
 * Return: An XArray internal entry corresponding to this value.
 */
static inline void *xa_mk_internal(unsigned long v)
{
	return (void *)((v << 2) | 2);
}

// XA_RETRY_ENTRY计算出来的值为1026
#define XA_RETRY_ENTRY		xa_mk_internal(256)

/**
 * xa_is_advanced() - Is the entry only permitted for the advanced API?
 * @entry: Entry to be stored in the XArray.
 *
 * Return: %true if the entry cannot be stored by the normal API.
 */
static inline bool xa_is_advanced(const void *entry)
{
	return xa_is_internal(entry) && (entry <= XA_RETRY_ENTRY);
}
```

`xa_is_node()`，判断是否为节点条目，节点条目的定义：内部条目并且值大于4096
```c
static inline bool xa_is_node(const void *entry)
{
	return xa_is_internal(entry) && (unsigned long)entry > 4096;
}
```

`xa_is_zero()`，判断是否为零条目
```c
// XA_ZERO_ENTRY，其值为1030
#define XA_ZERO_ENTRY		xa_mk_internal(257)

/**
 * xa_is_zero() - Is the entry a zero entry?
 * @entry: Entry retrieved from the XArray
 *
 * The normal API will return NULL as the contents of a slot containing
 * a zero entry.  You can only see zero entries by using the advanced API.
 *
 * Return: %true if the entry is a zero entry.
 */
static inline bool xa_is_zero(const void *entry)
{
	return unlikely(entry == XA_ZERO_ENTRY);
}
```

`xa_is_value()`，判断条目是值还是指针，通过条目的最后一位来进行判断
```c
// include/linux/xarray.h

/**
 * xa_is_value() - Determine if an entry is a value.
 * @entry: XArray entry.
 *
 * Context: Any context.
 * Return: True if the entry is a value, false if it is a pointer.
 */
static inline bool xa_is_value(const void *entry)
{
	return (unsigned long)entry & 1;
}
```


### 1.3 xarray API分析
1. `DEFINE_XARRAY(name)`，创建xarray
```c
// include/linux/xarray.h
// 创建一个xarray，此时xa_head指向NULL
#define DEFINE_XARRAY(name) DEFINE_XARRAY_FLAGS(name, 0)

#define DEFINE_XARRAY_FLAGS(name, flags)				\
	struct xarray name = XARRAY_INIT(name, flags)

#define XARRAY_INIT(name, flags) {				\
	// 初始化一个spin_lock
	.xa_lock = __SPIN_LOCK_UNLOCKED(name.xa_lock),		\
	.xa_flags = flags,					\
	.xa_head = NULL,					\
}
```

2. `xa_init()`，同`DEFINE_XARRAY(name)`作用一样，创建xarray
```c
// include/linux/xarray.h

static inline void xa_init(struct xarray *xa)
{
	xa_init_flags(xa, 0);
}

static inline void xa_init_flags(struct xarray *xa, gfp_t flags)
{
	spin_lock_init(&xa->xa_lock);
	xa->xa_flags = flags;
	xa->xa_head = NULL;
}
```

3. 添加键值对`xa_store()`
```c
// lib/xarray.c

// index表示索引，entry表示需要存储的数据
void *xa_store(struct xarray *xa, unsigned long index, void *entry, gfp_t gfp)
{
	void *curr;
	// 添加键值对时，需要对xarray->xa_lock进行上锁
	xa_lock(xa);
	curr = __xa_store(xa, index, entry, gfp);
	xa_unlock(xa);

	return curr;
}
EXPORT_SYMBOL(xa_store);

void *__xa_store(struct xarray *xa, unsigned long index, void *entry, gfp_t gfp)
{
	XA_STATE(xas, xa, index);
	void *curr;

	// 判断是否为高级条目，如果是直接返回
	if (WARN_ON_ONCE(xa_is_advanced(entry)))
		return XA_ERROR(-EINVAL);
	// 对于含有XA_FLAGS_TRACK_FREE标志的xarray
	// entry为空值时需要对其进行转换为XA_ZERO_ENTRY
	if (xa_track_free(xa) && !entry)
		entry = XA_ZERO_ENTRY;

	do {
		curr = xas_store(&xas, entry);
		if (xa_track_free(xa))
			xas_clear_mark(&xas, XA_FREE_MARK);
	} while (__xas_nomem(&xas, gfp));

	return xas_result(&xas, curr);
}
EXPORT_SYMBOL(__xa_store);

/**
 * xas_store() - Store this entry in the XArray.
 * @xas: XArray operation state.
 * @entry: New entry.
 *
 * If @xas is operating on a multi-index entry, the entry returned by this
 * function is essentially meaningless (it may be an internal entry or it
 * may be %NULL, even if there are non-NULL entries at some of the indices
 * covered by the range).  This is not a problem for any current users,
 * and can be changed if needed.
 *
 * Return: The old entry at this index.
 */
void *xas_store(struct xa_state *xas, void *entry)
{
	struct xa_node *node;
	// slot首先指向根节点的slot
	void __rcu **slot = &xas->xa->xa_head;
	unsigned int offset, max;
	int count = 0;
	int values = 0;
	void *first, *next;
	// 判断entry是值还是指针
	bool value = xa_is_value(entry);

	if (entry) {
		bool allow_root = !xa_is_node(entry) && !xa_is_zero(entry);
		// 创建slot并存储该条目
		first = xas_create(xas, allow_root);
	} else {
		first = xas_load(xas);
	}

	if (xas_invalid(xas))
		return first;
	node = xas->xa_node;
	if (node && (xas->xa_shift < node->shift))
		xas->xa_sibs = 0;
	if ((first == entry) && !xas->xa_sibs)
		return first;

	next = first;
	offset = xas->xa_offset;
	max = xas->xa_offset + xas->xa_sibs;
	if (node) {
		slot = &node->slots[offset];
		if (xas->xa_sibs)
			xas_squash_marks(xas);
	}
	if (!entry)
		xas_init_marks(xas);

	for (;;) {
		/*
		 * Must clear the marks before setting the entry to NULL,
		 * otherwise xas_for_each_marked may find a NULL entry and
		 * stop early.  rcu_assign_pointer contains a release barrier
		 * so the mark clearing will appear to happen before the
		 * entry is set to NULL.
		 */
		rcu_assign_pointer(*slot, entry);
		if (xa_is_node(next) && (!node || node->shift))
			xas_free_nodes(xas, xa_to_node(next));
		if (!node)
			break;
		count += !next - !entry;
		values += !xa_is_value(first) - !value;
		if (entry) {
			if (offset == max)
				break;
			if (!xa_is_sibling(entry))
				entry = xa_mk_sibling(xas->xa_offset);
		} else {
			if (offset == XA_CHUNK_MASK)
				break;
		}
		next = xa_entry_locked(xas->xa, node, ++offset);
		if (!xa_is_sibling(next)) {
			if (!entry && (offset > max))
				break;
			first = next;
		}
		slot++;
	}

	update_node(xas, node, count, values);
	return first;
}
EXPORT_SYMBOL_GPL(xas_store);

/*
 * xas_create() - Create a slot to store an entry in.
 * @xas: XArray operation state.
 * @allow_root: %true if we can store the entry in the root directly
 *
 * Most users will not need to call this function directly, as it is called
 * by xas_store().  It is useful for doing conditional store operations
 * (see the xa_cmpxchg() implementation for an example).
 *
 * Return: If the slot already existed, returns the contents of this slot.
 * If the slot was newly created, returns %NULL.  If it failed to create the
 * slot, returns %NULL and indicates the error in @xas.
 */
static void *xas_create(struct xa_state *xas, bool allow_root)
{
	struct xarray *xa = xas->xa;
	void *entry;
	void __rcu **slot;
	struct xa_node *node = xas->xa_node;
	int shift;
	unsigned int order = xas->xa_shift;

	if (xas_top(node)) {
		entry = xa_head_locked(xa);
		xas->xa_node = NULL;
		if (!entry && xa_zero_busy(xa))
			entry = XA_ZERO_ENTRY;
		shift = xas_expand(xas, entry);
		if (shift < 0)
			return NULL;
		if (!shift && !allow_root)
			shift = XA_CHUNK_SHIFT;
		entry = xa_head_locked(xa);
		slot = &xa->xa_head;
	} else if (xas_error(xas)) {
		return NULL;
	} else if (node) {
		unsigned int offset = xas->xa_offset;

		shift = node->shift;
		entry = xa_entry_locked(xa, node, offset);
		slot = &node->slots[offset];
	} else {
		shift = 0;
		entry = xa_head_locked(xa);
		slot = &xa->xa_head;
	}

	while (shift > order) {
		shift -= XA_CHUNK_SHIFT;
		if (!entry) {
			node = xas_alloc(xas, shift);
			if (!node)
				break;
			if (xa_track_free(xa))
				node_mark_all(node, XA_FREE_MARK);
			rcu_assign_pointer(*slot, xa_mk_node(node));
		} else if (xa_is_node(entry)) {
			node = xa_to_node(entry);
		} else {
			break;
		}
		entry = xas_descend(xas, node);
		slot = &node->slots[xas->xa_offset];
	}

	return entry;
}
```



### ida和idr原理
idr机制在linux内核中指的是整数ID管理机制，实质上来讲，是一种将一个整数ID号和一个指针关联在一起的机制，IDA是用IDR来实现的ID分配机制，与IDR的区别是IDA仅仅分配、管理ID，并不将ID和指针关联

ida和idr用于对内核中的整数资源的分配和管理，可以为内核中其他功能模块提供一种获取并占用唯一整数id的方法

最后一层的叶子节点的bitmap位图中的一个位代表一个整数id，用于表示该id是否已经被申请使用，1表示已使用，0表示空闲。


### 代码分析

ida数据结构
```c
// include/linux/types.h
typedef unsigned int __bitwise gfp_t;

// include/linux/xarray.h
struct xarray {
    spinlock_t      xa_lock;
/* private: The rest of the data structure is not to be used directly. */
    gfp_t           xa_flags;
    void __rcu *    xa_head;
};

// include/linux/idr.h

/*
 * IDA - ID Allocator, use when translation from id to pointer isn't necessary.
 */
#define IDA_CHUNK_SIZE		128	/* 128 bytes per chunk */
// IDA_BITMAP_LONGS大小为16
#define IDA_BITMAP_LONGS	(IDA_CHUNK_SIZE / sizeof(long))
// IDA_BITMAP_BITS大小为1024
#define IDA_BITMAP_BITS 	(IDA_BITMAP_LONGS * sizeof(long) * 8)

// ida_bitmap为一个长度16的unsigned long数组
struct ida_bitmap {
	unsigned long		bitmap[IDA_BITMAP_LONGS];
};

struct ida {
    struct xarray xa;
};
```

分配一个未使用过的整数id
```c
// include/linux/idr.h
/**
 * ida_alloc() - Allocate an unused ID.
 * @ida: IDA handle.
 * @gfp: Memory allocation flags.
 *
 * Allocate an ID between 0 and %INT_MAX, inclusive.
 *
 * Context: Any context. It is safe to call this function without
 * locking in your code.
 * Return: The allocated ID, or %-ENOMEM if memory could not be allocated,
 * or %-ENOSPC if there are no free IDs.
 */
static inline int ida_alloc(struct ida *ida, gfp_t gfp)
{
	return ida_alloc_range(ida, 0, ~0, gfp);
}

// lib/idr.c
/**
 * ida_alloc_range() - Allocate an unused ID.
 * @ida: IDA handle.
 * @min: Lowest ID to allocate.
 * @max: Highest ID to allocate.
 * @gfp: Memory allocation flags.
 *
 * Allocate an ID between @min and @max, inclusive.  The allocated ID will
 * not exceed %INT_MAX, even if @max is larger.
 *
 * Context: Any context. It is safe to call this function without
 * locking in your code.
 * Return: The allocated ID, or %-ENOMEM if memory could not be allocated,
 * or %-ENOSPC if there are no free IDs.
 */
int ida_alloc_range(struct ida *ida, unsigned int min, unsigned int max,
			gfp_t gfp)
{
	XA_STATE(xas, &ida->xa, min / IDA_BITMAP_BITS);
	unsigned bit = min % IDA_BITMAP_BITS;
	unsigned long flags;
	struct ida_bitmap *bitmap, *alloc = NULL;

	if ((int)min < 0)
		return -ENOSPC;

	if ((int)max < 0)
		max = INT_MAX;

retry:
	xas_lock_irqsave(&xas, flags);
next:
	bitmap = xas_find_marked(&xas, max / IDA_BITMAP_BITS, XA_FREE_MARK);
	if (xas.xa_index > min / IDA_BITMAP_BITS)
		bit = 0;
	if (xas.xa_index * IDA_BITMAP_BITS + bit > max)
		goto nospc;

	if (xa_is_value(bitmap)) {
		unsigned long tmp = xa_to_value(bitmap);

		if (bit < BITS_PER_XA_VALUE) {
			bit = find_next_zero_bit(&tmp, BITS_PER_XA_VALUE, bit);
			if (xas.xa_index * IDA_BITMAP_BITS + bit > max)
				goto nospc;
			if (bit < BITS_PER_XA_VALUE) {
				tmp |= 1UL << bit;
				xas_store(&xas, xa_mk_value(tmp));
				goto out;
			}
		}
		bitmap = alloc;
		if (!bitmap)
			bitmap = kzalloc(sizeof(*bitmap), GFP_NOWAIT);
		if (!bitmap)
			goto alloc;
		bitmap->bitmap[0] = tmp;
		xas_store(&xas, bitmap);
		if (xas_error(&xas)) {
			bitmap->bitmap[0] = 0;
			goto out;
		}
	}

	if (bitmap) {
		bit = find_next_zero_bit(bitmap->bitmap, IDA_BITMAP_BITS, bit);
		if (xas.xa_index * IDA_BITMAP_BITS + bit > max)
			goto nospc;
		if (bit == IDA_BITMAP_BITS)
			goto next;

		__set_bit(bit, bitmap->bitmap);
		if (bitmap_full(bitmap->bitmap, IDA_BITMAP_BITS))
			xas_clear_mark(&xas, XA_FREE_MARK);
	} else {
		if (bit < BITS_PER_XA_VALUE) {
			bitmap = xa_mk_value(1UL << bit);
		} else {
			bitmap = alloc;
			if (!bitmap)
				bitmap = kzalloc(sizeof(*bitmap), GFP_NOWAIT);
			if (!bitmap)
				goto alloc;
			__set_bit(bit, bitmap->bitmap);
		}
		xas_store(&xas, bitmap);
	}
out:
	xas_unlock_irqrestore(&xas, flags);
	if (xas_nomem(&xas, gfp)) {
		xas.xa_index = min / IDA_BITMAP_BITS;
		bit = min % IDA_BITMAP_BITS;
		goto retry;
	}
	if (bitmap != alloc)
		kfree(alloc);
	if (xas_error(&xas))
		return xas_error(&xas);
	return xas.xa_index * IDA_BITMAP_BITS + bit;
alloc:
	xas_unlock_irqrestore(&xas, flags);
	alloc = kzalloc(sizeof(*bitmap), gfp);
	if (!alloc)
		return -ENOMEM;
	xas_set(&xas, min / IDA_BITMAP_BITS);
	bit = min % IDA_BITMAP_BITS;
	goto retry;
nospc:
	xas_unlock_irqrestore(&xas, flags);
	kfree(alloc);
	return -ENOSPC;
}
EXPORT_SYMBOL(ida_alloc_range);
```