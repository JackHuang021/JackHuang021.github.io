

基于4.19.246内核分析


### 相关文档
1. Documentation/arm64/booting.rst


### 汇编基础
我们编译使用的GCC交叉编译器，汇编代码需要符合GNU语法

GNU汇编由一系列的语句组成，每行一条语句，每条语句有三个可选部分：`label: instruction @ comment`
+ label: 标号，表示地址位置，有些指令前面可能会有标号，这样就可以通过这个标号得到指令的位置
+ instruction: 汇编指令
+ comment: 注释内容

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

##### br指令



1. adrp: address page的简写，这里的page指的是大小为4KB的连续内存，根据PC的偏移地址计算目标页地址
```bash
adrp <xd>, <label>
```

内核启动入口，`arch/arm64/kernel/head.S`是内核的启动代码文件
```c
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
刚进入内核的时候是关闭了MMU和cache的，随后执行`stext`

```c
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
```c
/*
 * Preserve the arguments passed by the bootloader in x0 .. x3
 */
preserve_boot_args:
    // 将FDT物理地址暂存在x21寄存器中，释放x0
	mov	x21, x0				// x21=FDT

	adr_l	x0, boot_args			// record the contents of
	stp	x21, x1, [x0]			// x0 .. x3 at kernel entry
	stp	x2, x3, [x0, #16]

	dmb	sy				// needed before dc ivac with
						// MMU off

	mov	x1, #0x20			// 4 x 8 bytes
	b	__inval_dcache_area		// tail call
ENDPROC(preserve_boot_args)
```

