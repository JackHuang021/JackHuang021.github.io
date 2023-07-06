

基于4.19.246内核分析


### 相关文档
1. Documentation/arm64/booting.rst


### ARMv8体系架构
ARMv8架构继承了ARMv7与之前处理器的技术的基础，除了对现有的16/32bit的Thumb2指令支持外，也向前兼容了现有的A32（ARM 32bit）指令集，基于64bit的aarch64架构，除了新增A64（ARM 64bit）指令集外，也扩充了现有的A32和T32（Thumb2 32bit）指令集，另外还增加了CRYPTO加密模块支持

ARMv8提供了aarch32和aarch64两种执行状态（Execution State）

aarch32
+ 提供13个32bit通用寄存器R0-R12，一个32bit的PC指针（R15），堆栈指针SP（R13），链接寄存器LR（R14）
+ 提供1个32bit异常链接寄存器ELR，用于Hyp mode下的异常返回
+ 提供32个64bit SIMD向量和标量floating-point的支持
+ 提供两个指令集A32和T32
+ 兼容ARMv7的异常模型
+ 协处理器只支持CP10、CP11、CP14、CP15

aarch64
+ 提供31个64bit通用寄存器X0-X30(W0-W30 低32位)，其中X30是程序链接寄存器LR
+ 提供一个64bit的PC指针，堆栈指针SPx，异常链接寄存器ELRx
+ 提供32个128bit SIMD向量和标量floating-point支持
+ 定义ARMv8异常等级ELx（x范围为0-3），x越大等级越高，权限越大
+ 定义一组PSTATE，用来保存PE（Process Element）状态

aarch64执行状态下的寄存器
| 寄存器 | 说明 |
| :-: | :- |
| x0 | 用来保存返回值传参 |
| x1~x7 | 用来保存函数的传参 |
| x8 | 也可以用来保存返回值 |
| x9~x28 | 一般寄存器，无特殊用途 |
| x29（FP寄存器）| 用来保存栈底指针 |
| X30 (LR寄存器) | 用来保存返回地址 |
| X31 (SP寄存器) | 用来保存栈顶地址 |
| X32 (PC寄存器) | 用来保存当前执行指令的地址 |

ARMv8定义EL0~EL3共四个异常等级来控制PE的行为
+ 执行在EL0称为非特权执行
+ EL2没有secure state，只有none-secure state
+ EL3只有secure state，实现EL0/EL1的secure和non-secure之间的切换
+ EL0和EL1必须要实现，EL2/EL3是可选实现

### 汇编基础
ARM指令使用的是三地址码，格式如下：`<opcode> {<cond>} {S} <Rd>, <Rn>, <shifter_operand>`
+ opcode: 操作码，指令需要执行的操作类型
+ cond: 指令执行条件码
+ S: 条件码设置项，决定本次指令执行是否影响PSTATE寄存器响应状态位值
+ Rd: 目标寄存器，A64指令集可以选择X0~X30或者W0~W30
+ Rn: 第一个操作数的寄存器，和Rd一样
+ shift_operand: 第二个操作数，可以是立即数

cond项表明了指令的执行条件，每一条ARM指令都可以在规定的条件下执行，每条ARM指令包括4位的条件码，位于指令的最高4位[31:28]，条件码共有16种，每种条件码用2个字符表示，这两个字符可以添加到指令助记符的后面，与指令同时使用，当指令的执行条件满足时，指令才被执行，否则指令被忽略；如果指令后不写条件码，则使用默认条件AL（无条件执行）

### ARM汇编指令分类
| 类型 | 说明 |
| :-: | :- |
| 跳转指令 | 条件跳转、无条件跳转指令 |
| 异常产生指令 | 系统调用类指令（SVC HVC SMC）|
| 系统寄存器指令 | 读写系统寄存器 |
| 数据处理指令 | 包括各种算数运算、逻辑运算、位操作指令 |
| load/store内存访问指令 |
| 协处理器指令 | A64没有协处理器指令 |

#### 跳转指令
跳转指令用于实现程序流程的跳转，在ARM中有两种方法可以实现程序流程的跳转：
+ 直接向程序计数器PC写入跳转地址值，可以实现4GB的地址空间中的任意跳转
+ 使用专门的跳转指令，可以完成从当前指令向前或向后的32MB的地址空间的跳转

##### b指令
ARM处理器立即跳转到给定的目标地址，注意存储在跳转指令中的实际值是相对当前PC值的一个偏移量，并不是一个绝对地址，其为一个有效偏移位数为26位的有符号数（即PC位置前后32M的地址空间）
```c
b{cond} addr
```

##### bl指令
跳转之前会将下一条指令的地址存储在lr寄存器中，然后跳转到目标地址执行，可以通过将lr寄存器的内容重新加载到PC中，来返回到跳转指令之后的那个指令处执行，该指令是实现子程序调用的一个基本且常用的手段
```c
bl{cond} addr
```

##### blx指令
blx指令从ARM指令集跳转到指令中所指定的目标地址，并将处理器的工作状态由ARM状态切换到Thumb状态，同时将PC的当前内容保存到lr寄存器中，当子程序使用Thumb指令集，而调用者使用ARM指令集时，可以通过BLX指令实现子程序的调用和处理器工作状态的切换
```c
blx addr
```

##### bx指令
bx指令跳转到指令中所指定的目标地址，目标地址处的指令即可以是ARM指令，也可以是Thumb指令
```c
bl{cond} addr
```

##### br指令
绝对跳转指令，br（branch to register）指令执行的是无条件的绝对跳转，且跳转后不返回

##### blr指令
blr(branch with link to register)指令

#### 数据处理指令
##### cmp指令
cmp指令用于把一个寄存器的内容和另一个寄存器的内容或立即数进行比较，同时更新CPSR中条件标志位的值，标志位表示的是操作数1和操作数2的大小关系，该指令会将两个操作数进行一次减法运算，但不存储结果，只更改CPSR中条件标志位


#### 程序状态寄存器访问指令
##### msr指令
msr指令用于将操作数的内容传送到程序状态寄存器的特殊域中，其中操作数可以为通用寄存器或立即数
```c
msr{cond} 程序状态寄存器, 操作数
```

#### 加载、存储指令
##### stp指令
把一对寄存器写入到右边的内存，`stp w0,w1,[x2]`把w0和w1的值写入到右边内存

### 启动代码分析
内核启动入口，`arch/arm64/kernel/head.S`是内核的启动代码文件
```armasm
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET + TEXT_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
	__HEAD
_head:
	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
#ifdef CONFIG_EFI
	/*
	 * This add instruction has no meaningful effect except that
	 * its opcode forms the magic "MZ" signature required by UEFI.
	 */
	add	x13, x18, #0x16
	b	stext
#else
	b	stext				// branch to kernel start, magic
	.long	0				// reserved
#endif
	le64sym	_kernel_offset_le		// Image load offset from start of RAM, little-endian
	le64sym	_kernel_size_le			// Effective size of kernel image, little-endian
	le64sym	_kernel_flags_le		// Informative flags, little-endian
	.quad	0				// reserved
	.quad	0				// reserved
	.quad	0				// reserved
	.ascii	"ARM\x64"			// Magic number
#ifdef CONFIG_EFI
	.long	pe_header - _head		// Offset to the PE header.
```
刚进入内核的时候是关闭了各个level的MMU和cache的，随后执行`stext`


```armasm
// arch/arm64/kernel/head.S
ENTRY(stext)
	bl	preserve_boot_args
	bl	el2_setup			// Drop to EL1, w0=cpu_boot_mode
	adrp	x23, __PHYS_OFFSET
	and	x23, x23, MIN_KIMG_ALIGN - 1	// KASLR offset, defaults to 0
	bl	set_cpu_boot_mode_flag
	bl	__create_page_tables
	/*
	 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
	 * details.
	 * On return, the CPU will be ready for the MMU to be turned on and
	 * the TCR will have been set.
	 */
	bl	__cpu_setup			// initialise processor
	b	__primary_switch
ENDPROC(stext)
```

接下来看`preserve_boot_args`函数
```armasm
/*
 * Preserve the arguments passed by the bootloader in x0 .. x3
 */
preserve_boot_args:
    // 将FDT物理地址暂存在x21寄存器中，释放x0，X0在uboot启动后是存放设备树地址的
	mov	x21, x0				// x21=FDT

	// boot_args这个地址从哪里来的？
	adr_l	x0, boot_args			// record the contents of
	// 保存x21 x1的值到x0
	stp	x21, x1, [x0]			// x0 .. x3 at kernel entry
	// 保存x2 x3的值到x0 ?
	stp	x2, x3, [x0, #16]
	// ?
	dmb	sy				// needed before dc ivac with
						// MMU off

	// 将x1的值设置为0x20
	// boot_args变量长度是4x8bytes 32字节即0x20
	mov	x1, #0x20			// 4 x 8 bytes
	b	__inval_dcache_area		// tail call
ENDPROC(preserve_boot_args)
```
在调用`__inval_dcache_area`之前，x0是`boot_args`这段memory的首地址，x1是末尾地址的偏移，即0x20，arm64的boot protocol对启动时候的x0~x3这四个寄存器有严格的限制，x0是dtb的物理地址，x1~x3必须是0

接下来就到了`el2_setup`
```armasm
/*
 * If we're fortunate enough to boot at EL2, ensure that the world is
 * sane before dropping to EL1.
 *
 * Returns either BOOT_CPU_MODE_EL1 or BOOT_CPU_MODE_EL2 in w0 if
 * booted in EL1 or EL2 respectively.
 */
ENTRY(el2_setup)
	msr	SPsel, #1			// We want to use SP_EL{1,2}
	// 当前的exception level保存在PSTATE中
	// CurrentEL是获取PSTATE中current exception level域的特殊寄存器
	mrs	x0, CurrentEL
	// 判断是否处于EL2
	cmp	x0, #CurrentEL_EL2
	// 是EL2的话跳转到1f
	b.eq	1f
	// 执行到这里说明CPU处于EL1
	mov_q	x0, (SCTLR_EL1_RES1 | ENDIAN_SET_EL1)
	msr	sctlr_el1, x0
	mov	w0, #BOOT_CPU_MODE_EL1		// This cpu booted in EL1
	isb
	ret
	// 执行到这里说明CPU处于EL2
1:	mov_q	x0, (SCTLR_EL2_RES1 | ENDIAN_SET_EL2)
	msr	sctlr_el2, x0

#ifdef CONFIG_ARM64_VHE
	/*
	 * Check for VHE being present. For the rest of the EL2 setup,
	 * x2 being non-zero indicates that we do have VHE, and that the
	 * kernel is intended to run at EL2.
	 */
	mrs	x2, id_aa64mmfr1_el1
	ubfx	x2, x2, #8, #4
#else
	mov	x2, xzr
#endif

	/* Hyp configuration. */
	mov_q	x0, HCR_HOST_NVHE_FLAGS
	cbz	x2, set_hcr
	mov_q	x0, HCR_HOST_VHE_FLAGS
set_hcr:
	msr	hcr_el2, x0
	isb

	/*
	 * Allow Non-secure EL1 and EL0 to access physical timer and counter.
	 * This is not necessary for VHE, since the host kernel runs in EL2,
	 * and EL0 accesses are configured in the later stage of boot process.
	 * Note that when HCR_EL2.E2H == 1, CNTHCTL_EL2 has the same bit layout
	 * as CNTKCTL_EL1, and CNTKCTL_EL1 accessing instructions are redefined
	 * to access CNTHCTL_EL2. This allows the kernel designed to run at EL1
	 * to transparently mess with the EL0 bits via CNTKCTL_EL1 access in
	 * EL2.
	 */
	cbnz	x2, 1f
	mrs	x0, cnthctl_el2
	orr	x0, x0, #3			// Enable EL1 physical timers
	msr	cnthctl_el2, x0
1:
	msr	cntvoff_el2, xzr		// Clear virtual offset

#ifdef CONFIG_ARM_GIC_V3
	/* GICv3 system register access */
	mrs	x0, id_aa64pfr0_el1
	ubfx	x0, x0, #24, #4
	cbz	x0, 3f

	mrs_s	x0, SYS_ICC_SRE_EL2
	orr	x0, x0, #ICC_SRE_EL2_SRE	// Set ICC_SRE_EL2.SRE==1
	orr	x0, x0, #ICC_SRE_EL2_ENABLE	// Set ICC_SRE_EL2.Enable==1
	msr_s	SYS_ICC_SRE_EL2, x0
	isb					// Make sure SRE is now set
	mrs_s	x0, SYS_ICC_SRE_EL2		// Read SRE back,
	tbz	x0, #0, 3f			// and check that it sticks
	msr_s	SYS_ICH_HCR_EL2, xzr		// Reset ICC_HCR_EL2 to defaults

3:
#endif

	/* Populate ID registers. */
	mrs	x0, midr_el1
	mrs	x1, mpidr_el1
	msr	vpidr_el2, x0
	msr	vmpidr_el2, x1

#ifdef CONFIG_COMPAT
	msr	hstr_el2, xzr			// Disable CP15 traps to EL2
#endif

	/* EL2 debug */
	mrs	x1, id_aa64dfr0_el1		// Check ID_AA64DFR0_EL1 PMUVer
	sbfx	x0, x1, #8, #4
	cmp	x0, #1
	b.lt	4f				// Skip if no PMU present
	mrs	x0, pmcr_el0			// Disable debug access traps
	ubfx	x0, x0, #11, #5			// to EL2 and allow access to
4:
	csel	x3, xzr, x0, lt			// all PMU counters from EL1

	/* Statistical profiling */
	ubfx	x0, x1, #32, #4			// Check ID_AA64DFR0_EL1 PMSVer
	cbz	x0, 7f				// Skip if SPE not present
	cbnz	x2, 6f				// VHE?
	mrs_s	x4, SYS_PMBIDR_EL1		// If SPE available at EL2,
	and	x4, x4, #(1 << SYS_PMBIDR_EL1_P_SHIFT)
	cbnz	x4, 5f				// then permit sampling of physical
	mov	x4, #(1 << SYS_PMSCR_EL2_PCT_SHIFT | \
		      1 << SYS_PMSCR_EL2_PA_SHIFT)
	msr_s	SYS_PMSCR_EL2, x4		// addresses and physical counter
5:
	mov	x1, #(MDCR_EL2_E2PB_MASK << MDCR_EL2_E2PB_SHIFT)
	orr	x3, x3, x1			// If we don't have VHE, then
	b	7f				// use EL1&0 translation.
6:						// For VHE, use EL2 translation
	orr	x3, x3, #MDCR_EL2_TPMS		// and disable access from EL1
7:
	msr	mdcr_el2, x3			// Configure debug traps

	/* LORegions */
	mrs	x1, id_aa64mmfr1_el1
	ubfx	x0, x1, #ID_AA64MMFR1_LOR_SHIFT, 4
	cbz	x0, 1f
	msr_s	SYS_LORC_EL1, xzr
1:

	/* Stage-2 translation */
	msr	vttbr_el2, xzr

	cbz	x2, install_el2_stub

	mov	w0, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
	isb
	ret

install_el2_stub:
	/*
	 * When VHE is not in use, early init of EL2 and EL1 needs to be
	 * done here.
	 * When VHE _is_ in use, EL1 will not be used in the host and
	 * requires no configuration, and all non-hyp-specific EL2 setup
	 * will be done via the _EL1 system register aliases in __cpu_setup.
	 */
	mov_q	x0, (SCTLR_EL1_RES1 | ENDIAN_SET_EL1)
	msr	sctlr_el1, x0

	/* Coprocessor traps. */
	mov	x0, #0x33ff
	msr	cptr_el2, x0			// Disable copro. traps to EL2

	/* SVE register access */
	mrs	x1, id_aa64pfr0_el1
	ubfx	x1, x1, #ID_AA64PFR0_SVE_SHIFT, #4
	cbz	x1, 7f

	bic	x0, x0, #CPTR_EL2_TZ		// Also disable SVE traps
	msr	cptr_el2, x0			// Disable copro. traps to EL2
	isb
	mov	x1, #ZCR_ELx_LEN_MASK		// SVE: Enable full vector
	msr_s	SYS_ZCR_EL2, x1			// length for EL1.

	/* Hypervisor stub */
7:	adr_l	x0, __hyp_stub_vectors
	msr	vbar_el2, x0

	/* spsr */
	mov	x0, #(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT |\
		      PSR_MODE_EL1h)
	msr	spsr_el2, x0
	msr	elr_el2, lr
	mov	w0, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
	eret
ENDPROC(el2_setup)
```

