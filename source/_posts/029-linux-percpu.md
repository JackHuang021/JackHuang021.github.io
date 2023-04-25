---
title: Linux percpu机制
tags:
  - Linux
  - percpu
categories: Linux
abbrlink: cda25bee
date: 2022-12-30 10:17:56
---

#### percpu机制
在SMP架构中，每个CPU都拥有自己的高速缓存，通常，L1 cache是CPU独占的，每个CPU都有一份，它的速度自然是最快的，当CPU载入一个全局数据时，会逐级地查看高速缓存，如果没有在缓存中命中，就从内存中载入，并加入到各级cache中，当下次需要读取这个值时，直接读取cache将获得非常快的速度，比直接读取内存高出几个数量级。

对于读而言，cache带来了巨大的性能提升，当涉及到修改时，也是在缓存中进行操作，然后同步到内存中，对于单CPU而言，这没有什么问题，但是对于SMP架构而言，一个CPU对全局数据的修改将会导致所有其它CPU上对该全局数据的缓存失效，需要全部进行更新，这个操作将带来性能上的损失。

而最好的同步机制就是抑制同步产生的条件，因为每一种显式的同步机制都有着不可忽视的性能开销，percpu就是这样一种同步机制：为了避免多个CPU对全局数据的竞争而导致的性能损失，percpu直接为每个CPU生成一份独有的数据备份，每个数据备份占用独立的内存，CPU不应该修改不属于自己的这部分数据，这样就避免了多CPU对全局数据的竞争问题。

事实上，percpu只适合在特殊条件下使用：也就是当它确定在系统的CPU上的数据在逻辑上是独立的时候。正因为它的这种应用场景，CPU之间的percpu变量并不需要同步，在整个percpu的生命周期内，percpu变量对应的CPU副本都被对应的CPU独占使用。

#### percpu的存储
在linux内核的Image中，存在各种不同的section，在内核代码中也时常可以看到__attribute__关键字自定义section ：`__attribute__(section(".section"))`

`__attribute__(section())`的作用就是将所修饰的对象放在编译生成二进制文件的指定 section中，最常见的section有.data .bss，在程序链接的阶段将会确定每个section最终的加载地址

对于普通的变量而言，


#### percpu变量的使用

##### percpu的定义：静态定义和动态定义

静态地定义一个percpu变量
```c
DEFINE_PER_CPU(type, name);
// include/linux/percpu-defs.h
#define DEFINE_PER_CPU(type, name) \
    DEFINE_PER_CPU_SECTION(type, name, "")

#define DEFINE_PER_CPU_SECTION(type, name, sec) \
    __PCPU_ATTRS(sec) __typeof__(type) name

#define __PCPU_ATTRS(sec) \
    __percpu __attribute__((section(PER_CPU_BASE_SECTION sec))) \
    PER_CPU_ATTRIBUTES

// include/asm-generic/percpu.h
#define PER_CPU_BASE_SECTION ".data..percpu"

// 宏展开后为
#ifdef CONFIG_SMP
#define DEFINE_PER_CPU(type, name) \
    __percpu __attribute__((section(".data..percpu"))) __typeof__(type) name;\
#else \
    __percpu __attribute__((section(".data"))) __typeof__(type) name; \
#endif
```
整个定义翻译过来就是：在SMP架构下，被定义的percpu变量在编译后放在`.data..percpu`这个section中

##### percpu读写
对于静态定义的percpu变量，通常使用以下的接口
```c
get_cpu_var(name);
put_cpu_var(name);
```
使用通常是这样的
```c
DEFINE_PER_CPU(int, val);

int *p = &get_cpu_val(val);
*p++;
put_cpu_var(val);
```

事实上，put_cpu_var 并非字面上理解的：将变量放回内存，事实上它仅仅是使能了在 get_cpu_var 函数中关闭的内核抢占。所以，put_cpu_var()和get_cpu_var()是成对出现的，因为这段时间内核抢占处于关闭状态，他们之间的代码不宜执行太长时间。

动态percpu变量操作的接口
```c
// 动态percpu变量操作
per_cpu_ptr(ptr, cpu);
```
与静态变量的操作接口不一样，这个接口允许指定CPU，不再是只能获取当前CPU的值，而第一个参数是动态申请时返回的指针，通过该指针加上offset来获取percpu变量的地址，通常操作如下：
```c
int *ptr = alloc_percpu(int);

int *p = per_cpu_ptr(ptr, raw_smp_processor_id());
(*p)++;
```
其中，raw_smp_processor_id()函数返回当前CPU num，这个示例也就是操作当前CPU的percpu变量，这个接口并不需要禁止内核抢占，因为不管进程被切换到哪个CPU上执行，它所操作的都是第二个参数提供的CPU


#### percpu实现
##### percpu内存块描述数据结构
```c
struct pcpu_alloc_info {
    size_t  
};
```

##### percpu内存的初始化
percpu主要的初始化函数为`setup_per_cpu_areas()`，这个函数在`start_kernel()`中被直接调用
```c
// include/linux/percpu.h
#define PERCPU_MODULE_RESERVE   (8 << 10)

// mm/percpu.c
void __init setup_per_cpu_areas(void)
{
	unsigned long delta;
	unsigned int cpu;
	int rc;

	/*
	 * Always reserve area for module percpu variables.  That's
	 * what the legacy allocator did.
	 */
    // 申请并初始化内存块
	rc = pcpu_embed_first_chunk(PERCPU_MODULE_RESERVE,
				    PERCPU_DYNAMIC_RESERVE, PAGE_SIZE, NULL,
				    pcpu_dfl_fc_alloc, pcpu_dfl_fc_free);
	if (rc < 0)
		panic("Failed to initialize percpu areas.");

	delta = (unsigned long)pcpu_base_addr - (unsigned long)__per_cpu_start;
    // 计算每个cpu相对于源内存地址的偏移量
    // 当使用get_cpu_var(name)来获取变量地址时，首先找到的是源变量，然后通过地址偏移找到对应
    // cpu的percpu变量
	for_each_possible_cpu(cpu)
		__per_cpu_offset[cpu] = delta + pcpu_unit_offsets[cpu];
}


/**
 * pcpu_embed_first_chunk - embed the first percpu chunk into bootmem
 * @reserved_size: the size of reserved percpu area in bytes
 * @dyn_size: minimum free size for dynamic allocation in bytes
 * @atom_size: allocation atom size
 * @cpu_distance_fn: callback to determine distance between cpus, optional
 * @alloc_fn: function to allocate percpu page
 * @free_fn: function to free percpu page
 *
 * This is a helper to ease setting up embedded first percpu chunk and
 * can be called where pcpu_setup_first_chunk() is expected.
 *
 * If this function is used to setup the first chunk, it is allocated
 * by calling @alloc_fn and used as-is without being mapped into
 * vmalloc area.  Allocations are always whole multiples of @atom_size
 * aligned to @atom_size.
 *
 * This enables the first chunk to piggy back on the linear physical
 * mapping which often uses larger page size.  Please note that this
 * can result in very sparse cpu->unit mapping on NUMA machines thus
 * requiring large vmalloc address space.  Don't use this allocator if
 * vmalloc space is not orders of magnitude larger than distances
 * between node memory addresses (ie. 32bit NUMA machines).
 *
 * @dyn_size specifies the minimum dynamic area size.
 *
 * If the needed size is smaller than the minimum or specified unit
 * size, the leftover is returned using @free_fn.
 *
 * RETURNS:
 * 0 on success, -errno on failure.
 */
// 生成percpu内存块
// 保留大小为8k字节
// percpu动态分配区域大小为28k
// atom_size 12k
int __init pcpu_embed_first_chunk(size_t reserved_size, size_t dyn_size,
				  size_t atom_size,
				  pcpu_fc_cpu_distance_fn_t cpu_distance_fn,
				  pcpu_fc_alloc_fn_t alloc_fn,
				  pcpu_fc_free_fn_t free_fn)
{
	void *base = (void *)ULONG_MAX;
	void **areas = NULL;
	struct pcpu_alloc_info *ai;
	size_t size_sum, areas_size;
	unsigned long max_distance;
	int group, i, highest_group, rc = 0;

	ai = pcpu_build_alloc_info(reserved_size, dyn_size, atom_size,
				   cpu_distance_fn);
	if (IS_ERR(ai))
		return PTR_ERR(ai);

	size_sum = ai->static_size + ai->reserved_size + ai->dyn_size;
	areas_size = PFN_ALIGN(ai->nr_groups * sizeof(void *));

	areas = memblock_alloc(areas_size, SMP_CACHE_BYTES);
	if (!areas) {
		rc = -ENOMEM;
		goto out_free;
	}

	/* allocate, copy and determine base address & max_distance */
	highest_group = 0;
	for (group = 0; group < ai->nr_groups; group++) {
		struct pcpu_group_info *gi = &ai->groups[group];
		unsigned int cpu = NR_CPUS;
		void *ptr;

		for (i = 0; i < gi->nr_units && cpu == NR_CPUS; i++)
			cpu = gi->cpu_map[i];
		BUG_ON(cpu == NR_CPUS);

		/* allocate space for the whole group */
		ptr = alloc_fn(cpu, gi->nr_units * ai->unit_size, atom_size);
		if (!ptr) {
			rc = -ENOMEM;
			goto out_free_areas;
		}
		/* kmemleak tracks the percpu allocations separately */
		kmemleak_free(ptr);
		areas[group] = ptr;

		base = min(ptr, base);
		if (ptr > areas[highest_group])
			highest_group = group;
	}
	max_distance = areas[highest_group] - base;
	max_distance += ai->unit_size * ai->groups[highest_group].nr_units;

	/* warn if maximum distance is further than 75% of vmalloc space */
	if (max_distance > VMALLOC_TOTAL * 3 / 4) {
		pr_warn("max_distance=0x%lx too large for vmalloc space 0x%lx\n",
				max_distance, VMALLOC_TOTAL);
#ifdef CONFIG_NEED_PER_CPU_PAGE_FIRST_CHUNK
		/* and fail if we have fallback */
		rc = -EINVAL;
		goto out_free_areas;
#endif
	}

	/*
	 * Copy data and free unused parts.  This should happen after all
	 * allocations are complete; otherwise, we may end up with
	 * overlapping groups.
	 */
	for (group = 0; group < ai->nr_groups; group++) {
		struct pcpu_group_info *gi = &ai->groups[group];
		void *ptr = areas[group];

		for (i = 0; i < gi->nr_units; i++, ptr += ai->unit_size) {
			if (gi->cpu_map[i] == NR_CPUS) {
				/* unused unit, free whole */
				free_fn(ptr, ai->unit_size);
				continue;
			}
			/* copy and return the unused part */
			memcpy(ptr, __per_cpu_load, ai->static_size);
			free_fn(ptr + size_sum, ai->unit_size - size_sum);
		}
	}

	/* base address is now known, determine group base offsets */
	for (group = 0; group < ai->nr_groups; group++) {
		ai->groups[group].base_offset = areas[group] - base;
	}

	pr_info("Embedded %zu pages/cpu s%zu r%zu d%zu u%zu\n",
		PFN_DOWN(size_sum), ai->static_size, ai->reserved_size,
		ai->dyn_size, ai->unit_size);

	pcpu_setup_first_chunk(ai, base);
	goto out_free;

out_free_areas:
	for (group = 0; group < ai->nr_groups; group++)
		if (areas[group])
			free_fn(areas[group],
				ai->groups[group].nr_units * ai->unit_size);
out_free:
	pcpu_free_alloc_info(ai);
	if (areas)
		memblock_free_early(__pa(areas), areas_size);
	return rc;
}
```

##### section的处理与链接
在链接脚本arch/arm64/kernel/vmlinux.lds中，对`.data..percpu section`进行了重定位，为该section 分别加载地址
```
. = ALIGN((1 << 12)); 
.data..percpu : AT(ADDR(.data..percpu) - 0) { 
    __per_cpu_load = .; 
    __per_cpu_start = .; 
    *(.data..percpu..first) . = ALIGN((1 << 12)); 
    *(.data..percpu..page_aligned) . = ALIGN((1 << (6))); 
    *(.data..percpu..read_mostly) . = ALIGN((1 << (6))); 
    *(.data..percpu) *(.data..percpu..shared_aligned) __per_cpu_end = .; 
}
```

使用alloc_percpu动态地分配一个percpu变量，返回percpu变量的地址，但是这个返回的地址并非是可以直接使用的变量地址，就像静态定义的那样，这只是一个原始数据，真正被使用的数据被copy成n份分别保存在每个CPU独占的地址空间中，在访问percpu变量时就是对每个副本进行访问
```c
type __percpu *ptr alloc_percpu(type);
```
