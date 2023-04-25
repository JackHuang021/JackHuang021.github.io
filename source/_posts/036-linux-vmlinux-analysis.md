---
title: arm64 vmlinux.lds分析
date: 2023-01-28 16:02:15
tags:
  - Linux
  - vmlinux
categories: Linux
---

vmlinux.lds.S主要是用来组织内核每个函数存放在内核镜像文件的位置，编译内核源码生成内核文件的过程分两步，一个是编译过程，另一个是链接过程。vmlinux.lds.S要做的就是告诉编译器如何链接编译好的各个.o文件，未经编译的内核源码是不存在vmlinux.lds链接脚本的，在`arch/arm64/kernel`目录只有vmlinux.lds.S文件，以及在`include/asm-generic`目录有一个与之关联的vmlinux.lds.h文件，在内核编译的时候会根据一些宏定义和传入的参数构建出针对特定平台、特定架构的vmlinux.lds链接脚本

### 链接器
现代软件工程中，一个大的程序通常都由多个源文件组成，其中包含以高级计算机语言编写的源文件以及汇编语言编写的汇编文件。在编译构建过程中会分别对这些源文件进行汇编或者编译并生成目标文件，这些目录文件包含代码段、数据段、符号表等内容。而链接则是把这些目标文件的代码段、数据段以及符号表等内容收集起来并按照某种格式（例如ELF）组合成一个可执行二进制文件的过程。而这个过程是使用**链接器**来完成的。

链接器在链接过程中会使用到一个链接脚本文件，该文件用于描述链接的过程，当没有通过“-T”参数指定链接脚本时，链接器会使用内置的链接脚本。链接脚本控制着如何把输入文件中的段合并到输出文件的段中，以及这些段的地址空间布局等。本质上则是把在编译构建过程中大量的二进制文件（.o文件）合并成一个可执行的二进制文件。

#### 常用段说明
| 段名称 | 主要作用 |
| :-: | :-: |
| init段 | Linux定义的一种初始化过程中才会用到的段，一旦初始化完成，那么这些段所占用的内存会被释放掉 |
| text段 | 代码段，通常是指用来存放程序代码的一块内存区域，这部分区域的大小在程序运行前就已经确定 |
| data段 | 数据段，通常是指用来存放程序中已初始化的全局变量的一块内存区域，数据段属于静态内存分配 |
| bss段 | 通常是指用来存放程序中未初始化的全局变量和静态变量的一块内存区域，属于静态内存分配 |

#### 地址解释
**运行地址：** CPU执行一条程序中指令时的执行地址，也就是PC寄存器中的值。更简单的讲，就是要寻址到一个指令或者变量所使用的地址  
**加载地址：** 程序中指令和变量等加载到RAM上的地址
**链接地址：** 链接过程中链接器为指令和变量分配的地址

**地址之间的联系**
+ 如果没有打开MMU，并且使用的是位置相关设计，那么加载地址、运行地址、链接地址三者需要一致
+ 当打开MMU之前，如果使用的是位置无关设计，那么运行地址和加载地址应该是一致的

### 链接脚本分析
链接脚本最终要把.o文件生成最终的二进制可执行文件，也就是把每一个.o文件整合到一个大文件中，这个大文件有总的text、data、bss段，还有一些别的段

示例
```c
SECTIONS
(
    . = 0x10000;        /* 链接地址为0x10000，即指定了后面text段的地址 */
    .text = {*(.text)}  /* 输出文件的text段内容，由所有文件的text段组成（* 理解为所有的.o文件）*/
    . = 0x8000000;      /* 链接地址改变为0x8000000，指定了后面data段的链接地址 */
    .data = {*(.data)}  /* 输出文件的data段由所有输入文件的data段组成 */
    .bss = {*(.bss)}    /* 输出文件的bss段由所有输入文件的bss段组成 */
)
```

链接脚本语法：
+ 在链接脚本中，有一个特殊的符号：“.”，用于表示当前位置计数器。在vmlinux.lds.S文件中很多地方都会使用到
+ 在链接脚本中有一个常用的编程技巧：为每个段（或者多个段）设置一些符号，用于标识内存位置的开始和结束，这样便可以在C语言代码中访问每个段（或者多个段）的起始地址和结束地址

#### vmlinux.lds.S文件
linux内核针对ARM64架构的链接脚本放置于`arch/arm64/kernel/vmlinux.lds.S`文件中：
```c
/* SPDX-License-Identifier: GPL-2.0 */
/*
 * ld script to make ARM Linux kernel
 * taken from the i386 version by Russell King
 * Written by Martin Mares <mj@atrey.karlin.mff.cuni.cz>
 */

#define RO_EXCEPTION_TABLE_ALIGN	8
#define RUNTIME_DISCARD_EXIT

#include <asm-generic/vmlinux.lds.h>
#include <asm/cache.h>
#include <asm/hyp_image.h>
#include <asm/kernel-pgtable.h>
#include <asm/memory.h>
#include <asm/page.h>

#include "image.h"

OUTPUT_ARCH(aarch64)
/* 设置程序入口为_text */
ENTRY(_text)

/* 定义jiffies参数值，jiffies_64定义在kernel/time/timer.c中 */
jiffies = jiffies_64;


#ifdef CONFIG_KVM
#define HYPERVISOR_EXTABLE					\
	. = ALIGN(SZ_8);					\
	__start___kvm_ex_table = .;				\
	*(__kvm_ex_table)					\
	__stop___kvm_ex_table = .;

#define HYPERVISOR_PERCPU_SECTION				\
	. = ALIGN(PAGE_SIZE);					\
	HYP_SECTION_NAME(.data..percpu) : {			\
		*(HYP_SECTION_NAME(.data..percpu))		\
	}
#else /* CONFIG_KVM */
#define HYPERVISOR_EXTABLE
#define HYPERVISOR_PERCPU_SECTION
#endif

#define HYPERVISOR_TEXT					\
	/*						\
	 * Align to 4 KB so that			\
	 * a) the HYP vector table is at its minimum	\
	 *    alignment of 2048 bytes			\
	 * b) the HYP init code will not cross a page	\
	 *    boundary if its size does not exceed	\
	 *    4 KB (see related ASSERT() below)		\
	 */						\
	. = ALIGN(SZ_4K);				\
	__hyp_idmap_text_start = .;			\
	*(.hyp.idmap.text)				\
	__hyp_idmap_text_end = .;			\
	__hyp_text_start = .;				\
	*(.hyp.text)					\
	HYPERVISOR_EXTABLE				\
	__hyp_text_end = .;

#define IDMAP_TEXT					\
	. = ALIGN(SZ_4K);				\
	__idmap_text_start = .;				\
	*(.idmap.text)					\
	__idmap_text_end = .;

#ifdef CONFIG_HIBERNATION
#define HIBERNATE_TEXT					\
	. = ALIGN(SZ_4K);				\
	__hibernate_exit_text_start = .;		\
	*(.hibernate_exit.text)				\
	__hibernate_exit_text_end = .;
#else
#define HIBERNATE_TEXT
#endif

#ifdef CONFIG_UNMAP_KERNEL_AT_EL0
#define TRAMP_TEXT					\
	. = ALIGN(PAGE_SIZE);				\
	__entry_tramp_text_start = .;			\
	*(.entry.tramp.text)				\
	. = ALIGN(PAGE_SIZE);				\
	__entry_tramp_text_end = .;
#else
#define TRAMP_TEXT
#endif

/*
 * The size of the PE/COFF section that covers the kernel image, which
 * runs from _stext to _edata, must be a round multiple of the PE/COFF
 * FileAlignment, which we set to its minimum value of 0x200. '_stext'
 * itself is 4 KB aligned, so padding out _edata to a 0x200 aligned
 * boundary should be sufficient.
 */
PECOFF_FILE_ALIGNMENT = 0x200;

#ifdef CONFIG_EFI
#define PECOFF_EDATA_PADDING	\
	.pecoff_edata_padding : { BYTE(0); . = ALIGN(PECOFF_FILE_ALIGNMENT); }
#else
#define PECOFF_EDATA_PADDING
#endif

/* SECTIONS{}是链接脚本语法中的关键命令，用于描述输出文件的内存布局 */
/* SECTIONS命令告诉链接文件如何把输入文件的段映射到输出文件的各个段中 */
SECTIONS
{
	/*
	 * XXX: The linker does not define how output sections are
	 * assigned to input sections when there are multiple statements
	 * matching the same input section name.  There is no documented
	 * order of matching.
	 */
    /* /DISCARD/是一个特殊的输出段，该段引入的任何输入段将不会出现在输出文件中 */
	DISCARDS
	/DISCARD/ : {
		*(.interp .dynamic)
		*(.dynsym .dynstr .hash .gnu.hash)
	}

    /* 把代码段的链接地址设置为KIMAGE_VADDR */
	. = KIMAGE_VADDR;

    /* HEAD_TEXT在include/asm-generic/vmlinux.lds.h中定义 */
    /* #define HEAD_TEXT  KEEP(*(.head.text)) */
    /* .head.text输出段，对应输入段为HEAD_TEXT（本质为*(.head.text)） */
    /* 意思是将所有目标文件中的.head.text段放入.head.text输出段中 */
    /* _text = .; 用于标识_text段的开始 */
	.head.text : {
		_text = .;
		HEAD_TEXT
	}

    /* 代码段，会汇集目标文件中的多个输入段到.text中 */
	.text : {			/* Real text segment		*/
		_stext = .;		/* Text and read-only data	*/
			IRQENTRY_TEXT
			SOFTIRQENTRY_TEXT
			ENTRY_TEXT
			TEXT_TEXT
			SCHED_TEXT
			CPUIDLE_TEXT
			LOCK_TEXT
			KPROBES_TEXT
			HYPERVISOR_TEXT
			IDMAP_TEXT
			HIBERNATE_TEXT
			TRAMP_TEXT
			*(.fixup)
			*(.gnu.warning)
		. = ALIGN(16);
		*(.got)			/* Global offset table		*/
	}

	/*
	 * Make sure that the .got.plt is either completely empty or it
	 * contains only the lazy dispatch entries.
	 */
	.got.plt : { *(.got.plt) }
	ASSERT(SIZEOF(.got.plt) == 0 || SIZEOF(.got.plt) == 0x18,
	       "Unexpected GOT/PLT entries detected!")

	. = ALIGN(SEGMENT_ALIGN);
	_etext = .;			/* End of text section */

	/* everything from this point to __init_begin will be marked RO NX */
  	RO_DATA(PAGE_SIZE)

	idmap_pg_dir = .;
	. += IDMAP_DIR_SIZE;
	idmap_pg_end = .;

#ifdef CONFIG_UNMAP_KERNEL_AT_EL0
	tramp_pg_dir = .;
	. += PAGE_SIZE;
#endif

#ifdef CONFIG_ARM64_SW_TTBR0_PAN
	reserved_ttbr0 = .;
	. += RESERVED_TTBR0_SIZE;
#endif
	swapper_pg_dir = .;
	. += PAGE_SIZE;
	swapper_pg_end = .;

	. = ALIGN(SEGMENT_ALIGN);
	__init_begin = .;
	__inittext_begin = .;

	INIT_TEXT_SECTION(8)

	__exittext_begin = .;
	.exit.text : {
		EXIT_TEXT
	}
	__exittext_end = .;

	. = ALIGN(4);
	.altinstructions : {
		__alt_instructions = .;
		*(.altinstructions)
		__alt_instructions_end = .;
	}

	. = ALIGN(SEGMENT_ALIGN);
	__inittext_end = .;
	__initdata_begin = .;

	.init.data : {
		INIT_DATA
		INIT_SETUP(16)
		INIT_CALLS
		CON_INITCALL
		INIT_RAM_FS
		*(.init.rodata.* .init.bss)	/* from the EFI stub */
	}
	.exit.data : {
		EXIT_DATA
	}

	PERCPU_SECTION(L1_CACHE_BYTES)
	HYPERVISOR_PERCPU_SECTION

	.rela.dyn : ALIGN(8) {
		*(.rela .rela*)
	}

	__rela_offset	= ABSOLUTE(ADDR(.rela.dyn) - KIMAGE_VADDR);
	__rela_size	= SIZEOF(.rela.dyn);

#ifdef CONFIG_RELR
	.relr.dyn : ALIGN(8) {
		*(.relr.dyn)
	}

	__relr_offset	= ABSOLUTE(ADDR(.relr.dyn) - KIMAGE_VADDR);
	__relr_size	= SIZEOF(.relr.dyn);
#endif

	. = ALIGN(SEGMENT_ALIGN);
	__initdata_end = .;
	__init_end = .;

	_data = .;
	_sdata = .;
	RW_DATA(L1_CACHE_BYTES, PAGE_SIZE, THREAD_ALIGN)

	/*
	 * Data written with the MMU off but read with the MMU on requires
	 * cache lines to be invalidated, discarding up to a Cache Writeback
	 * Granule (CWG) of data from the cache. Keep the section that
	 * requires this type of maintenance to be in its own Cache Writeback
	 * Granule (CWG) area so the cache maintenance operations don't
	 * interfere with adjacent data.
	 */
	.mmuoff.data.write : ALIGN(SZ_2K) {
		__mmuoff_data_start = .;
		*(.mmuoff.data.write)
	}
	. = ALIGN(SZ_2K);
	.mmuoff.data.read : {
		*(.mmuoff.data.read)
		__mmuoff_data_end = .;
	}

	PECOFF_EDATA_PADDING
	__pecoff_data_rawsize = ABSOLUTE(. - __initdata_begin);
	_edata = .;

	BSS_SECTION(0, 0, 0)

	. = ALIGN(PAGE_SIZE);
	init_pg_dir = .;
	. += INIT_DIR_SIZE;
	init_pg_end = .;

	. = ALIGN(SEGMENT_ALIGN);
	__pecoff_data_size = ABSOLUTE(. - __initdata_begin);
	_end = .;

	STABS_DEBUG
	DWARF_DEBUG
	ELF_DETAILS

	HEAD_SYMBOLS

	/*
	 * Sections that should stay zero sized, which is safer to
	 * explicitly check instead of blindly discarding.
	 */
	.plt : {
		*(.plt) *(.plt.*) *(.iplt) *(.igot .igot.plt)
	}
	ASSERT(SIZEOF(.plt) == 0, "Unexpected run-time procedure linkages detected!")

	.data.rel.ro : { *(.data.rel.ro) }
	ASSERT(SIZEOF(.data.rel.ro) == 0, "Unexpected RELRO detected!")
}

#include "image-vars.h"

/*
 * The HYP init code and ID map text can't be longer than a page each,
 * and should not cross a page boundary.
 */
ASSERT(__hyp_idmap_text_end - (__hyp_idmap_text_start & ~(SZ_4K - 1)) <= SZ_4K,
	"HYP init code too big or misaligned")
ASSERT(__idmap_text_end - (__idmap_text_start & ~(SZ_4K - 1)) <= SZ_4K,
	"ID map text too big or misaligned")
#ifdef CONFIG_HIBERNATION
ASSERT(__hibernate_exit_text_end - (__hibernate_exit_text_start & ~(SZ_4K - 1))
	<= SZ_4K, "Hibernate exit text too big or misaligned")
#endif
#ifdef CONFIG_UNMAP_KERNEL_AT_EL0
ASSERT((__entry_tramp_text_end - __entry_tramp_text_start) == PAGE_SIZE,
	"Entry trampoline text too big")
#endif
/*
 * If padding is applied before .head.text, virt<->phys conversions will fail.
 */
ASSERT(_text == KIMAGE_VADDR, "HEAD is misaligned")
```


