---
title: ARMv8体系架构
tags:
  - ARMv8
  - Register
categories: ARM
abbrlink: 17165b72
date: 2023-01-18 14:22:33
---

### ARMv8-A架构及其对应处理器特性
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128094507.png)

### ARMv8异常等级
**软件运行异常级别**：
+ EL0: 普通用户应用程序
+ EL1: 操作系统内核通常被描述为具有特权的
+ EL2: 管理程序
+ EL3: 低级固件，包括安全监视器
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128095322.png)

ARMv8体系架构提供两种安全状态：secure和non-secure，每种安全状态有独立的物理地址空间范围，在Secure state，PE可以访问Secure和Non-secure的物理地址空间范围，在Non-secure state，PE只能访问Non-secure的物理地址空间范围，Secure state和Non-secure state将运行环境划分为Normal world和Secure world，且EL3存在于Secure state

异常等级切换规则：
+ 当发生一个异常时，异常等级只能上升或者维持不变
+ 当从一个异常返回时，异常等级只能下降或者维持不变
+ 一个异常要进入的异常等级称作target exception level，EL0不能作为target exception level，也就是说所有异常都不会在EL0中处理

### ARMv8执行状态
ARMv8体系结构提供了2种执行状态，AArch64和AArch32，其中AArch32执行状态用于实现与ARMv7体系结构兼容

执行状态定义了PE的执行环境，包括：
+ 寄存器宽度
+ 指令集
+ 异常模式
+ 虚拟内存系统架构
+ 编程模型

AArch64执行状态：
+ 提供31个64位的通用寄存器（X0~X30）
+ 提供64位的程序计数寄存器PC，栈指针寄存器SP，异常链接寄存器ELR
+ 提供32个128位的用于SIMD与浮点运算寄存器
+ 提供A64指令集
+ 使用ARMv8异常模型，支持4个异常等级（EL0~EL3）
+ 支持64位的虚拟内存寻址
+ 使用一组处理器状态（PSTATE）寄存器保存PE状态
+ 每个系统寄存器命名时，带有异常等级后缀，该后缀决定了可访问该寄存器的最低异常等级

AArch32执行状态
+ 提供13个32位的通用寄存器
+ 提供32位的程序计数寄存器PC、栈指针寄存器SP、链接寄存器LR，其中LR同时也用作ELR
+ 提供32个64位的用于SIMD和浮点运算的寄存器
+ 提供A32和T32指令集
+ 支持ARMv7-A异常模型，实现时将PE模式映射到ARMv8的异常模型
+ 支持32位的虚拟地址寻址
+ 使用一组处理器状态（PSTATE）寄存器保存PE状态，A32和T32指令通过APSR和CPSR访问

执行状态切换：
+ 如果需要在一个64位操作系统上运行32位的应用程序，就需要将执行状态从AArch64切换到AArch32
+ 当32位应用程序运行完成时，或者需要陷入64位操作系统执行时，就需要将执行状态从AArch32再切换回AArch64

执行状态切换规则：只能通过异常陷入更高的异常等级，才能进行执行状态的切换，例如在64位操作系统（EL1）上需要运行32位和64位应用程序（EL0），假设当前32位应用程序正在运行，那么他需要通过SVC指令或者接收到一个中断从而陷入到EL1的64位操作系统。此时操作系统可以进行任务调度，从而切换到64位的应用程序运行

### ARMv8寄存器组
AArch64的执行状态提供了可在所有时间和所有异常级别访问的31个64位通用寄存器，分别是X0~X30，每一个64位的通用寄存器(X0~X30)的低32位又由32位的寄存器(W0~W30)组成
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128094339.png)

#### AArch64特殊寄存器
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128102622.png)

当访问Zero寄存器，所有的写入都被忽略，所有的读取都返回0，AArch32执行状态使用WZR指令访问zero寄存器，使用WSP指令访问当前栈指针寄存器；AArch64执行状态使用XZR访问zero寄存器，使用SP指令访问当前栈指针寄存器，AArch32和AArch64执行状态下都使用PC指令访问程序计数寄存器
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128104400.png)

每个异常级别中都有专用的SP寄存器，退出该异常级别时不需要保存
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128104005.png)

**Zero寄存器**  
当Zero寄存器当作源寄存器读取时，会得到0值，当Zero寄存器当作目的寄存器写入时，写入的值被丢弃，

**Stack Pointer寄存器**  
在ARMv8架构中，每个异常级别都拥有栈指针寄存器，即拥有4个栈指针寄存器。在默认情况下，异常级别ELn对应SP_ELn，当处理器的执行状态为AArch64且不处于异常级别EL0，则可以使用与异常级别相关的专用64位堆栈指针(SP_ELn)和异常级别EL0相关的堆栈指针寄存器(SP_EL0)，各个异常级别与可使用的栈寄存器关系如下图所示（后缀t表示使用SP_EL0堆栈指针，h表示使用SP_ELx堆栈指针）
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128134348.png)

**Program Counter寄存器**  
ARMv8架构中删除了对PC寄存器的直接访问，PC寄存器不能作为命名寄存器访问，而是通过确定的指令隐含的访问

**Exception Link寄存器**  
ELR寄存器保存了异常返回地址，ARMv8定义了3个ELR寄存器，分别对应异常级别EL1, EL2, EL3，当异常发生时，异常返回地址将被保存在target exception level的ELR寄存器，当异常返回时，将使用的ELR寄存器中的值恢复到PC寄存器，和SPSR寄存器一样，异常发生时也只能使用与targe exception level相应的ELR_ELn

**Saved Process Status寄存器**  
当异常发生时，处理器的状态将会被保存到相关的SPSR寄存器中，异常发生后，在处理异常之前，处理器会自动的将PSTATE寄存器的内容保存到SPSR中，异常返回时，会将SPSR保存的处理器状态恢复到PSTATE中，ARMv8定义的SPSR寄存器如下，兼容ARMv7中的SPSR寄存器，只使用低32位。在ARMv8架构中，有3个SPSR寄存器，分别为SPSR_EL1、SPSR_EL2、SPSR_EL3，使用那个SPSR寄存器依赖于异常级别
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128135732.png)
各个位的定义如下：
+ N：符号位
+ Z：0标志
+ C：操作进位
+ V：溢出标志
+ SS：用于软件调试，异常发生的时候，可通过设置启用单步调试机制
+ IL：不合法的执行状态，保存自PSTATE.IL
+ D：处理器状态调试掩码，指示是否屏蔽来自观察点、断电和软件单步调试事件的调试异常
+ A：系统错误掩码
+ I：IRQ掩码位
+ F：FIQ掩码位
+ M[4]：发生异常时处理器的执行状态，0表示AArch64
+ M[3:0]：M[3:2]发生异常的级别，M[1]保留，M[0]根据此选择栈指针寄存器，0表示t，1表示h

#### Processor State
AArch64没有类似ARMv7 Current Program Status Register（CPSR）的寄存器，在AArch64的执行状态中，处理器状态要用PSTATE描述，但PSTATE不是寄存器，而是处理器状态各个位域的总称，PSTATE的大部分位域和传统CPSR寄存器中的位域相同。EL0可以访问PSTATE的N、Z、C、V位域，DAIF标志位需要经过配置对EL0才可见，其他域只能在EL1及更高的异常级别中访问。PSTATE的位域如下图所示：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230128144328.png)

PSTATE状态位的访问：ARMv8体系架构提供了一组特殊寄存器，用于访问PSTATE状态位

#### System Registers
ARMv8体系架构中定义了很多系统寄存器，通过访问和设置这些系统寄存器（通过MRS / MSR指令）来完成对处理器的功能设置，<register_name>_ELn，其中ELn标识了可访问该寄存器的最低异常等级

AMRv8体系架构支持如下7类系统寄存器
1. 通用系统控制寄存器
2. 调试寄存器
3. 性能监控寄存器
4. 活动监控寄存器
5. 统计扩展寄存器
6. 通用定时器寄存器

