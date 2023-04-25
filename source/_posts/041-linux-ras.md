---
title: Linux RAS
date: 2023-03-08 09:15:21
tags:
---

#### 文档编写规则
+ Rule：定义概念或术语或描述一个实现
+ Information：提供信息和指导，以便理解规范
+ Rationale：声明规范的规定方式
+ Implementation note：为规范的实施提供指导
+ Software usage：软件如何使用规范

#### 相关概念
Reliability, Availability, Serviceability(RAS)：
+ Reliability：可靠性
+ Availability：可用性
+ Serviceability：错误修正能力

correct service：当服务实现系统功能时，将提供正确的服务，包含：
+ 产生正确的结果
+ 在分配给任务的时间内产生结果
+ 不泄露安全信息

**软件运行异常级别**：
+ EL0: 普通用户应用程序
+ EL1: 操作系统内核通常被描述为具有特权的
+ EL2: 管理程序
+ EL3: 低级固件，包括安全监视器



APEI（ACPI Platform Error Interfaces）：为错误信息从固件到OS传递提供一个标准接口

SDEI（Software Delegated Exception Interface）：

### RAS Extension
##### 基本定义
PE（Processing Element）：将处理器处理事务的抽象过程定义为PE，称为处理元素，可以将PE简单理解为处理器核心

SIMD指令（Single Instruction Multiple Data）：单指令多数据流，SIMD能复制多个操作数，并把它们打包在大型寄存器的一组指令集。以同步的方式，在同一时间内执行同一条指令，SIMD特别适合于多媒体应用数据密集型运算。

FEAT_RAS：表示RAS Extension特性
RAS Extension是否实现：在AArch64状态，查询ID_AA64PFR0_EL1.RAS；在AArch32状态，查询ID_PFR0.RAS，RAS Extensipn在Armv8.2中是必须实现的

PE error state：当处理单元有异常或错误发生时的状态

RAS Extension扩展了一些异常寄存器，可以使PE当错误或异常发生时上报PE错误状态

ESB：Error Sysnchronization Barrier instruction, 错误同步屏障

RAS Extension增加了错误同步事件和错误同步屏障（ESB）指令

RAS Extensio定义了对于RAS的系统寄存器，包含可以访问错误记录的寄存器，这些系统寄存器都在contex-A的参考手册中有定义

为RAS服务的三个特性：
+ FEAT_IESB 特性：在异常进入和异常返回时插入隐形错误同步事件，这个特性受SCTLR_ELx.IESB这个位控制，IESB位会被加到ESR_ELx寄存器上，这个特性只在ARMv8.2上实现，这个特性只支持AArch64执行状态，ID_AA64MMFR2_EL1.IESB这个位指示该特性是否实现
+ FEAT_RASv1p1特性：扩展RAS系统寄存器以对RAS系统架构的支持
+ FEAT_DoubleFault特性：可在EL3异常等级中更改同步外部中止异常的路由并将SError中断视为不可屏蔽

RAS扩展没有规定PE中的可靠性、可用性和可服务性级别。PE包含的RAS功能，例如检测、纠正、包含或延迟错误，是实现定义的。RAS扩展定义了在PE中构建RAS功能的框架

RA扩展的一些术语，用于描述系统非正常工作下的几种错误状态：
+ failure：偏离系统正常运行的事件，包括数据损坏、数据丢失、服务丢失
+ error：偏离系统正常运行，错误的不正确的值已经造成损坏
+ fault：产生error的原因

在没有错误检测的系统中，所有error都是潜在错误，并由组件悄悄传播，直到它们被屏蔽或导致故障

IMPLEMENTATION DEFINED：表示基础架构未定义该行为

SError：是异步异常的一种，一般是来自External aborts，例如当memory system访问时，bus上产生的external aborts

如果满足以下任一条件，则将异常描述为异步的：
+ 不是因为直接执行某条指令而产生异常
+ 异常处理程序的返回地址不可以表明导致该异常的指令
+ 异常是不准确的

##### 错误处理
in-band错误响应：也称为外部中止（external aborts），为了避免和External abort异常区分，这里使用in-band错误响应描述PE对内存访问的响应

当PE访问内存或其他状态时，可能会在该内存或状态中检测到错误，并对检测到的错误进行纠正、延迟或者向PE发出信号，一个延迟的错误可能会向PE发送in-band错误响应，也可以通过其他方式发信号给PE

当组件在PE的读取或者维护缓存操作中检测到错误时：
+ 如果错误可以被纠正，它将被更正并返回更正后的数据
+ 如果错误无法纠正但可以被推迟，则它将被推迟
+ 如果错误无法纠正且在组件上实现和启用，则检测的错误将作为in-band错误响应向PE发出信号

组件（component）会记录检测到的错误并生成一个错误处理中断和错误恢复中断，组件会向RSA系统架构节点（node）上报错误，这个节点会记录错误也可能会生成错误处理中断，错误恢复中断或严重错误中断，这些取决于node的特性和配置


##### 错误传播

PE状态包括：
+ 通用、SIMD&FP、SVE寄存器
+ 系统寄存器
+ 特殊寄存器
+ PSTATE（ARMv8架构中，使用PSTATE来描述当前处理器的状态信息）

PE对错误的处理：
+ 错误处理指令提交到PE状态
+ 错误发生在取指令上，错误处理的指令已经提交执行
+ 错误发生在为提交的加载、存储或指令提取执行的转换表遍历中

对于PE，错误传播适用于PE在PE状态与任何其他PE状态或内存之间传播检测到的错误

错误由PE通过以下方式传播，如果故障未被激活则不允许进行传播：
+ 任何指令使用错误值将错误传播到目标：
    + 存储错误值
    + 将错误值写入到系统寄存器或PSTATE
+ 发生任何不该发生的操作：
    + 不允许的加载、转换表遍历或指令提取，包括来自硬件推测或预取的那些
    + 存储到一个错误的地址或不被允许的存储
    + 不被允许的间接或直接对系统寄存器的写入
    + 断言任何不会被断言的信号，例如中断
+ 需要发生的操作却没有执行
+ PE产生一个不明确的异常，而不是响应错误本身对应的异常
+ PE 丢弃它保存的处于修改状态的数据
+ 所需的单处理器语义、顺序或一致性的任何其他损失。

PE按照流程顺序异步地接受一个错误异常：
+ 错误值从第一个位置返回到通用寄存器
+ PE禁止将寄存器存储到第二个内存位置。特别是，位置没有更新，因此保留了它以前的值
+ 产生错误异常
+ 在出现错误异常时，架构施加的排序约束没有被违反，特别是那些与步骤2中存储的可观察性相关的约束

当以下所有条件都满足时，PE传播的错误才会由PE静默传播
1. 传播不是PE在接受错误生成的错误异常时所需操作的一部分
2. 传播不是PE执行同步错误的ESB指令所需操作的一部分
3. 该错误不会作为检测到的错误或延迟错误向consumer发出信号
4. 错误值保存在通用寄存器、SIMD&FP 或 SVE 寄存器以外的地方。
5. 在采取由错误产生的错误异常或执行同步错误的 ESB 指令之前，错误按程序顺序由指令传播，并传播到通用、SIMD&FP 或 SVE 寄存器之外
6. 错误不是通过将错误值作为输入操作数使用但在其他方面表现正确的指令传播的

案例1：PE产生了一个错误异常中断，并返回了一个错误值到通用寄存器，在发生错误异常之前，错误不会静默地传播到通用寄存器之外

PE认为以下两种情况都不是属于错误静默传播
+ 错误异常导致ESR_ELx，ELR_ELx和SPSR_ELx寄存器产生更新，这个是属于PE正常操作的一部分
+ 在错误异常产生后，软件将通用寄存器的值写入内存，这不会作为延迟错误向内存发出信号

如果以下任一示例附加操作发生在2和3之间，则PE已静默传播错误：

一个实现可能在PE状态本身中包含错误检测逻辑。当PE在PE状态中检测到错误时，使用该状态的指令会处理该错误，并且PE会生成IMPLEMENTATION DEFINED错误异常，作为SError中断异常。在这种情况下，实现PE的处理器包括一个RAS系统架构节点，该节点实现记录这些错误的错误记录

#### 错误异常的产生
当检测到的错误作为对架构执行的内存访问或缓存维护操作的in-band错误响应向PE发出信号时，将生成错误异常。这包括任何显式数据访问、指令获取、转换表遍历或由架构执行的指令对转换表进行的硬件更新。

错误异常被视为异步SError中断、同步External Data Abort异常或同步 External Instruction Abort 异常

由硬件推测或PE预取产生的错误是否可以生成错误异常由实现定义

#### 获取错误异常
如果实现了 FEAT_DoubleFault，那么对于所有非推测性的错误异常都将作为同步外部中止异常：
+ 取指令
+ 页表转换和翻译表的硬件更新指令获取

当以下异常被获取，PE在异常寄存器组中记录PE错误状态
+ 在AArch64执行状态同步外部中止被获取
+ 在AArch64和AArch32中SError中断异常被获取

当同步外部中止进入AArch32状态时，PE不会记录PE错误状态，异常类型和目标执行状态决定了PE可以记录的PE错误状态值集

##### PE error状态在异常寄存器组上的记录
PE在ESR_ELx中记录PE Error stat总共有7种状态
+ Uncontainable (UC)
+ Unrecoverable state (UEU)
+ Recoverable state (UER)
+ Restartable state (UEO)
+ Corrected (CE)
+ Uncategorized error
+ IMPLEMENTATION DEFINED syndrome

当异步SError中断在AArch64运行态上被获取，PE在ESR_ELx异常寄存器上记录PE错误状态
+ Uncategorized error（未分类错误）：ESR_ELx.ISS置为0，同时将ESR_ELx.IDS和ESR_ELx.DFSC置为0
+ IMPLEMENTATION DEFINDED syndrome：将ESR_ELx.IDS置为1
+ 其他的PE error state均记录在ESR_ELx.AET，通过ESR_ELx.IDS置0和ESR_ELx.DFSC为非零故障代码，表示ESR_ELx.AET是有效的

当同步外部停止异常发生在AArch64运行状态，PE会将PE error state记录在ESR_ELx.SET中，有以下三种状态：
+ Uncontainable(UC)
+ Recoverable state(UER)
+ Restartable state(UEO)

##### PE error state分类
PE在产生异常时的一些特定实现属性：
+ 错误是否是通过PE静默传播
+ PE 是否确定软件能够在发生异常的位置恢复执行


#### PE中的错误记录
在RAS系统架构中可以记录错误的组件被称为节点(node)，一个节点可以实现一个或多个错误记录

实现 RAS 扩展的 PE 可能实现节点的系统寄存器接口，节点的系统寄存器接口不限于仅访问PE节点

当错误被PE节点记录后，根据节点的配置，以下的一条或几条可能会生成
+ 错误处理中断
+ 错误恢复中断
+ 严重错误中断
+ in-band错误响应

##### 系统寄存器中的错误记录
如果节点的系统寄存器实现了，则可以从系统寄存器访问节点的错误记录，ERRIDR_EL1和ERRIDR这两个寄存器决定了可以从系统寄存器访问的最大错误记录数

AArch64的错误记录系统寄存器是ERX*_EL1系列

为了访问一个错误记录，需要以下步骤：
1. 设置错误选择寄存器，ERRSELR_EL1.SEL或者ERRSELR.SEL，为了取得需要访问错误记录的索引
2. 在ERX*_EL1或ERX*系统寄存器上访问错误记录

### RAS系统架构

#### Nodes
node：RAS中的一个node表示系统部件记录一个或多个系统组件检测到或使用的错误

RAS系统架构的实现包含了一个或多个node，RAS系统架构不要求系统中的所有组件都实现 RAS 系统架构或作为节点出现。

RAS系统架构不规定系统的可靠性、可用性和可服务性级别。系统包含的RAS功能，例如检测、纠正、包含或延迟错误，是实现定义的。Arm 建议将所有错误报告给 RAS 系统架构节点以启用错误恢复和故障处理。

RAS系统架构为node定义了以下特性：
1. 错误检测和纠正
2. 错误处理中断
3. 纠正错误计数
4. 错误记录时间戳
5. 错误恢复中断
6. 严重错误中断
7. 错误记录

每个node包含至少一个错误记录，也可以包含多个错误记录：
+ 在不同的错误记录中记录不同类型的错误
+ 将来自不同组件或组件访问的不同 FRU 的错误记录在不同的错误记录中

一组错误记录由一个或多个节点的错误记录组成

#### 检测和处理错误
组件检测和处理错误的时机：
+ 错误可以被纠正：
    + 纠正错误
    + 将错误报给node，node记录为一个纠正的错误，触发错误处理中断

### ATF文档里面对RAS的描述
这个文档描述了 Trusted Firmware-A 对RAS扩展的一个支持

结合EHF（Error Handling Framework），在（firmware-first handling）固件层面首先对错误进行处理，在non-secure中由错误产生的异常在EL3中进行处理

这些错误有四种：
+ Synchronous External Abort（SEA），同步外部中止
+ Asynchronous External Abort（SErrors），异步外部中止
+ Fault Handling，错误处理
+ Error Recovery interrupts，错误恢复中断

对于TFA来说，firmware-first handling表示异步异常适当路由到EL3，然后runtime firmware（BL31）中的一些软件组件会处理这些异常在EL3异常等级，这些组件会进行如下处理：
+ 在EL3中接收并处理这些异常，异常将在EL3中终止
+ 接收异常，部分在EL3中进行处理，其余的在较低EL secure中进行处理
+ 接收异常，部分在EL3中进行处理，其余的在较低EL secure中进行处理，另一部分在normal world也会参与一部分处理

ATF中的RAS可以在EL3中路由或处理由平台错误引起的异常，它允许平台定义外部中止处理程序，并注册 RAS 节点和中断，RAS框架还可以访问RAS扩展中引入的标准错误记录，这些记录都记录在RAS系统寄存器中

在runtime firmware中支持RAS扩展需要在编译时将选项RAS_EXTENSION设置为1，EL3_EXCEPTION_HANDLING和HANDLE_EA_EL3_FIRST_NS也要设置成1，RAS_TRAP_NS_ERR_REC_ACCESS选项控制Non-secure是否可以访问RAS错误记录寄存器

#### 在ATF中使用RAS框架
启用RAS支持的三个编译选项：
+ RAS_EXTENSION=1：在runtime firmware中包含RAS
+ EL3_EXCEPTION _HANDLING=1：使能在EL3中处理异常
+ HANDLE_EA_EL3_FIRST_NS=1：

#### 注册RAS错误节点
RAS节点（node）是系统中一个组件，它可以通过三个机制向PE发送错误：SEAs，SErrors，中断；RAS节点包含一个或多个错误记录，这些错误记录都存在寄存器里面，节点通过这些寄存器通告所通知错误的各种属性，ARM建议错误记录以一个标准错误记录格式呈现，RAS体系结构允许通过系统或内存映射寄存器访问错误记录。

不同平台需要枚举这些错误记录以下列之一的方式：
+ 用于探测错误记录的处理程序
+ 当探测识别出一个错误时，一个处理程序来处理它
+ 当内存映射的错误记录了，需要知晓内存基地址，对于系统寄存器记录的错误，需要知晓记录的起始索引和该索引的连续记录数；

上面这些信息提供后，runtime firmware接收到错误通知后，RAS框架可以遍历错误记录并探测错误，并调用适当的处理程序来处理它。

RAS框架提供宏去填充这些错误信息，这些宏根据其参数创建一个类型为struct err_record_info的结构，该结构稍后会传递给探测和错误处理程序
```c
// memory-mapped error 
ERR_RECORD_MEMMAP_V1(base_addr, size_num_k, probe, handler, aux)

// system register record error
ERR_RECORD_SYSREG_V1(idx_start, num_idx, probe, handler, aux)
```

当平台枚举了这些标准错误记录：
+ 返回非零值当错误被检测成一个标准错误记录
+ 检测到错误时将probe_data设置为错误记录的索引

#### 注册RAS中断
RAS节点可以通过错误处理或错误恢复中断发送错误到PE，固件优先处理这些中断，不同平台必须要设置和注册中断。

对于每一个RAS中断，平台提供了一个`struct ras_interrupt`，包含：
+ 中断号
+ 相关的错误记录信息，指向`struct err_record_info`
+ 另外的一些缓存信息

平台需要定义一个`struct ras_interrupt`数组，使用`REGISTER_RAS_INTERRUPT()`注册到RAS框架上，`struct ras_interrupt`数组必须按照中断号递增的顺序排列

#### Double-fault处理



### ATF相关
带ATF的芯片,通常的上电启动流程是：
BOOTROM—>PL（PreLoader）—>ATF—>optee—>uboot—>OS

### Linux EDAC驱动框架
EDAC模块的目标是检测并且上报系统错误

edac_mc处理Memory Correctable Errors(CE)和Memory Uncorrectable Errors(UE)

edac_device处理非内存设备的错误


### D2000V编程手册资料
D2000V检测的错误类型：
+ 可纠错误
+ 不可纠错误

D2000V支持一下两种报错机制：
+ 错误处理中断
+ 错误恢复中断

不同的错误类型可以通过不同的中断报错
+ 可纠错误只能通过错误处理中断报错
+ 不可纠错误既可以走错误处理中断报错，也可以走错误恢复中断报错，或者同时触发两种中断

为了验证软件处理流程的正确性，可使用错误注入寄存器模拟错误中断上报：
+ 对于不可纠错误，是检测即上报，往错误注入寄存器写入对应的错误编号就可以产生错误恢复中断上报
+ 对于可纠错误，是溢出后上报，可以有两种方式模拟报错：
    + 写对应错误编号的错误记录杂项寄存器0(ERR< n >MISC0)为0xFF00000000，然后往错误注入寄存器写入对应的错误编号，可以产生错误处理中断上报
    + 往错误注入寄存器连续写入255次对应的错误编号，可以产生错误处理中断上报


### RAS处理流程

#### 中断处理
1. 先从0xFC8 ERRDEVID(Device Configuration Register)寄存器读取该组错误记录的最大编号
2. 再从0xE00 ERRSGR(Error Group Status Register)寄存器读取错误记录的状态，其中bit[0:55]中bit(m)为1表示error_record(m)存在一个或多个错误
3. 接着对0x20 + 64*n Err(n)MISC0寄存器进行读操作，对BIT39进行判断可纠错误计数是否溢出

#### 错误注入


```bash
echo 7 > /proc/sys/kernel/printk
devmem2 0x32B2807C w 1
```

#### ras_event中定义的trace event
```c
trace_extlog_mem_event();
trace_mc_event();
trace_arm_event();
trace_non_standard_event();
trace_aer_event();
trace_memory_failure_event();
```





