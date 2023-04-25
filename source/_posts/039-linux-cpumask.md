---
title: Linux CPU状态管理
tags:
  - Linux
  - coumask
categories: Linux
---

#### cpumask定义
内核使用cpumask来记录CPU的状态，cpumask提供了系统中CPU集合的位图，一个bit代表了一个CPU的状态

cpumask结构体
```c

// include/linux/threads.h
// 在E2000 5.10内核中配置为256，即最大支持256个CPU
#define NR_CPUS     CONFIG_NR_CPUS

// include/linux/bits.h
#define BITS_PER_BYTE       8

// include/uapi/linux/kernel.h
#define __KERNEL_DIV_ROUND_UP(n, d) (((n) + (d) - 1) / (d))

// include/linux/kernel.h
#define DIV_ROUND_UP __KERNEL_DIV_ROUND_UP

// include/linux/bitops.h
#define BITS_PER_TYPE(type) (sizeof(type) * BITS_PER_BYTE)
#define BITS_TO_LONGS(nr)   DIV_ROUND_UP(nr, BITS_PER_TYPE(long))
#define BITS_TO_BYTES(nr)   DIV_ROUND_UP(nr, BITS_PER_TYPE(char))

// include/linux/types.h
#define DECLARE_BITMAP(name, bits) \
        unsigned long name[BITS_TO_LONGS(bits)]]

// include/linux/cpumask.h
// 实际就是定义一个足够大的unsigned long数组，其中数据中的每个bit表示CPU的一个状态
typedef struct cpumask { DECLARE_BITMAP{bits, NR_CPUS}; } cpumask_t;

// 转换后
typedef struct cpumask {
    unsigned long bits[BITS_TO_LONGS(NR_CPUS)];
} cpumask_t;

typedef struct cpumask cpumask_var_t[1];
```

#### cpumask接口

##### to_cpumask
这里就是强转为cpumask类型
```c
#define to_cpumask(bitmap) \
	((struct cpumask *)(1 ? (bitmap) \
		: (void *)sizeof(__check_is_bitmap(bitmap))))
```

##### cpumask_test_cpu
查看cpu对应位的状态
```c
static inline int cpumask_test_cpu(int cpu, const struct cpumask *cpumask)
{
	return test_bit(cpumask_check(cpu), cpumask_bits((cpumask)));
}
```

##### cpumask_next
`unsigned int cpumask_next(int n, const struct cpumask *srcp)`，查找下一个状态非0的CPU标号
```c
// include/linux/cpumask.h
#define nr_cpumask_bits     ((unsigned int)NR_CPUS)

// include/linux/bitmap.h
#define BITMAP_FIRST_WORD_MASK(start) (~0UL << ((start) & (BITS_PER_LONG - 1)))

// include/linux/kernel.h
#define __round_mask(x, y)  ((__typeof__(x))((y) - 1))
/**
 * round_down - round down to next specified power of 2
 * @x: the value to round
 * @y: multiple to round down to (must be a power of 2)
 *
 * Rounds @x down to next multiple of @y (which must be a power of 2).
 * To perform arbitrary rounding down, use rounddown() below.
 */
#define round_down(x, y) ((x) & ~__round_mask(x, y))

// include/asm-generic/bitops/builtin-__ffs.h
// 返回第一个非0bit的位置
static __always_inline unsigned long __ffs(unsigned long word)
{
    // 这个内建函数作用是返回输入数二进制表示从最低位开始(右起)的连续的0的个数
    return __builtin_ctzl(word);
}

// lib/find_bit.c
/*
 * This is a common helper function for find_next_bit, find_next_zero_bit, and
 * find_next_and_bit. The differences are:
 *  - The "invert" argument, which is XORed with each fetched word before
 *    searching it for one bits.
 *  - The optional "addr2", which is anded with "addr1" if present.
 */
static unsigned long _find_next_bit(const unsigned long *addr1,
		const unsigned long *addr2, unsigned long nbits,
		unsigned long start, unsigned long invert, unsigned long le)
{
	unsigned long tmp, mask;

	if (unlikely(start >= nbits))
		return nbits;
    // 获取start对应bit在数组所在位置的数据
	tmp = addr1[start / BITS_PER_LONG];
	if (addr2)
		tmp &= addr2[start / BITS_PER_LONG];
	tmp ^= invert;

	/* Handle 1st word. */
    // mask就是start到高64位为1，其余低于start的位置为0
	mask = BITMAP_FIRST_WORD_MASK(start);
	if (le)
		mask = swab(mask);
    
    // 获取start到高64位的数据
	tmp &= mask;

	start = round_down(start, BITS_PER_LONG);

	while (!tmp) {
		start += BITS_PER_LONG;
		if (start >= nbits)
			return nbits;

		tmp = addr1[start / BITS_PER_LONG];
		if (addr2)
			tmp &= addr2[start / BITS_PER_LONG];
		tmp ^= invert;
	}

	if (le)
		tmp = swab(tmp);

	return min(start + __ffs(tmp), nbits);
}

// 从offset开始查找下一个非0bit的位置
unsigned long find_next_bit(const unsigned long *addr, unsigned long size,
			    unsigned long offset)
{
	return _find_next_bit(addr, NULL, size, offset, 0UL, 0);
}
EXPORT_SYMBOL(find_next_bit);


// lib/cpumask.c
// 查找下一个状态非0的CPU标号
unsigned int cpumask_next(int n, const struct cpumask *srcp)
{
    if (n != -1)
        cpumask_check(n);
    return find_next_bit(cpumask_bits(srcp), nr_cpumask_bits, n + 1);
}
```

##### for_each_cpu
for_each_cpu(cpu, mask)，从cpu_mask中遍历所有非0bit的CPU
```c
#define for_each_cpu(cpu, mask) \
    for ((cpu) = -1; \
        (cpu) = cpumask_next((cpu), (mask)), \
        (cpu) < nr_cpu_ids;)
```

##### cpumask_copy
```c
// 实际就是对cpumask里面long数组的拷贝
static inline void cpumask_copy(struct cpumask *dstp,
						const struct cpumask *srcp)
{
	bitmap_copy(cpumask_bits(dstp), cpumask_bits(srcp), nr_cpumask_bits);
}
```

##### cpumask_of
```c
#define cpumask_of(cpu) (get_cpu_mask(cpu))

static inline const struct cpumask *get_cpu_mask(unsigned int cpu)
{
	const unsigned long *p = cpu_bit_bitmap[1 + cpu % BITS_PER_LONG];
	p -= cpu / BITS_PER_LONG;
	return to_cpumask(p);
}
```



