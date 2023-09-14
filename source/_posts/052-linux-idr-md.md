---
title: 052-linux-idr.md
date: 2023-09-14 14:03:48
tags:
  - Linux
  - idr
categories: Linux
---

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
#define IDA_BITMAP_LONGS	(IDA_CHUNK_SIZE / sizeof(long))
#define IDA_BITMAP_BITS 	(IDA_BITMAP_LONGS * sizeof(long) * 8)

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