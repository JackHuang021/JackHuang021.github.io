---
title: 077-linux-rcu
tags:
---


## 1. RCU介绍
RCU(Read-Copy Update)是数据同步的一种方式，RCU主要针对的数据对象是链表，目的是提高遍历读取数据的效率，使用RCU机制读取数据的时候不对链表进行耗时的加锁操作，这样在同一时间可以有多个线程可以读取该链表，对链表进行修改的时候需要进行加锁操作。RCU适用于需要频繁的读取数据，而相应修改数据不是很多的情景。

RCU将数据更新分为**Removal(删除)**和**Reclamation(回收)**两个阶段
+ Removal阶段：创建一个数据副本对齐进行修改，然后通过指针替换的方式将数据结构中的指针指向新版数据，这个阶段旧数据被标记为“待回收”，它的内存没有立即释放，而是通过RCU机制等待回收。
+ Reclamation阶段：在确保所有对旧数据的访问者都已经退出临界区之后回收旧数据。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241129100547.png)

## 2. RCU数据回收的宽限期
RCU宽限期（Grace Period）：RCU的回收机制依赖于“宽限期”，即在更新完成后，必须等待一个宽限期来确保所有“Pre-existing”读者（即在更新操作开始之前就已经进入临界区的读者，在更新阶段之后的读者已经访问的是新数据，无需等待其退出）已经退出。这段时间的长度取决于系统中的并发读者数量和它们的退出时机。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241129100351.png)

上图中，Reader和Writer并发执行，当Updater执行Removal操作完成后调用synchronize_rcu()，标志着更新结束，开始进入宽限期等待进行回收。这里并不需要考虑Reader-1和Reader-2退出读临界区，因为这两个读者已经是读到的最新数据。

## 2.1 如何检测是否结束宽限期

RCU通过内核中的定时器中断来定期检查当前是否有正在进行中的读者。在每次中断触发时会检查是否有新的读者进入临界区，并更新宽限期，如果没有任何读者活动，RCU就可以在宽限期结束时回收旧数据。如果有读者仍在临界区内，RCU会继续等待，直到所有读者退出。

## 3. RCU stall的原因
RCU stall（卡顿或停顿）问题通常指的是RCU在等待宽限期（Grace Period）结束时遇到的延迟问题，导致内核中更新操作的回收阶段无法按时完成。RCU stall 会影响系统性能，导致系统的更新操作、资源回收和内存释放出现延迟。这个问题与以下常见的几个原因相关：
1. 读者临界区长时间未退出
2. 长时间禁用CPU中断、抢占
3. 大量的中断导致rcu stall

当宽限期超过 `/sys/module/rcupdate/parameters/rcu_cpu_stall_timeout` 的值的时候，就会打印rcu stalled的消息。

当前我们系统里面rcu stall的超时时间为21s
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241129142320.png)

## 4. RCU stall测试
根据rcu stall的产生原理，编写内核测试驱动，在定时器回调函数中模拟了一个长时间禁用中断的情况。

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/slab.h>
#include <linux/kthread.h>
#include <linux/rcupdate.h>
#include <linux/delay.h>
#include <linux/timer.h>

static struct timer_list stall_timer;

static void rcu_stall_simulation(struct timer_list *t)
{
	unsigned long flags;

	pr_info("simulation rcu stall.\n");
	local_irq_save(flags);
	mdelay(30000);
	local_irq_restore(flags);
	pr_info("rcu stall simulation ended.\n");
}

static int __init rcu_stall_test_init(void)
{
	pr_info("rcu stall simulation driver loaded.\n");

	timer_setup(&stall_timer, rcu_stall_simulation, 0);
	mod_timer(&stall_timer, jiffies + msecs_to_jiffies(1000));

	return 0;
}

static void __exit rcu_stall_test_exit(void)
{
	pr_info("rcu stall simulation driver unloaded.\n");
	del_timer_sync(&stall_timer);
}

module_init(rcu_stall_test_init);
module_exit(rcu_stall_test_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("rcu stall test driver");
```

驱动加载后的日志如下图：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241129144232.png)

rcu stall相关的日志为：
```bash
[  169.856347] rcu: INFO: rcu_preempt detected stalls on CPUs/tasks:
[  169.862452] rcu: 	1-...0: (1 GPs behind) idle=e964/1/0x4000000000000002 softirq=9112/9113 fqs=1786
[  169.871408] rcu: 	(detected by 0, t=5252 jiffies, g=12745, q=3032 ncpus=4)
[  169.878277] Task dump for CPU 1:
[  169.881497] task:swapper/1       state:R  running task     stack:0     pid:0     ppid:1      flags:0x0000000a
[  169.891410] Call trace:
[  169.893848]  __switch_to+0xc8/0x13c
[  169.897342]  0x0
```

`[  169.862452] rcu: 	1-...0: (1 GPs behind) idle=e964/1/0x4000000000000002 softirq=9112/9113 fqs=1786`
+ `1-...0: (1 GPs behind)`：表示 CPU 1 上有一个进程或任务正在导致 RCU stall（被标记为“落后”1个 GPs，即“Grace Periods”
+ `softirq=9112/9113`：显示该 CPU 上处理的RCU软件中断的状态。第一个数字表示从系统启动以来到上个宽限期开始处理的RCU软中断数量，第二个数字表示系统启动以来RCU软中断处理完成的数量。如果它们之间的差异很大，可能表示软中断堆积，导致了长时间无法退出临界区。如果打印的rcu stall信息中后者的数字一直保持不变，则可能是中断长时间被禁用

`[  169.871408] rcu: 	(detected by 0, t=5252 jiffies, g=12745, q=3032 ncpus=4)`

+ `detected by 0`：表示 CPU 0 检测到了 RCPU stall。
+ `t=5252 jiffies`：表示从宽限期开始到现在已持续了 5252 个 jiffies，换算为时间即21s左右
+ `g=12745`：表示宽限期的序列号
+ `q=3032`：表示所有CPU排队的RCU回调函数总数
+ `ncpus=4`：表示系统中有 4 个 CPU 被监控

### 4.1 调试RCU stall问题

#### 4.1.1 初步定位是由哪个驱动模块导致

先初步定位可能引起rcu stall问题的驱动，可以临时禁用该驱动来验证是否是该驱动引发的问题

#### 4.1.2 利用rcu_cpu_stall_cputime参数排查

打开rcu_cpu_stall_cputime=1参数，内核在出现rcu stall问题时，会打印从上个宽限期开始时出现rcu stall的CPU的中断执行次数、上下文切换次数以及中断和task的cputime。可以利用这些信息排查是由什么原因导致的rcu stall，例如禁用中断、禁用中断底半部、禁用抢占、大量的硬件中断。

下面是4个典型的rcu stall场景：
1. 中断禁用
```bash
rcu:          hardirqs   softirqs   csw/system
rcu:  number:        0          0            0
rcu: cputime:        0          0            0   ==> 2500(ms)
```

2. 中断底半部禁用
```bash
rcu:          hardirqs   softirqs   csw/system
rcu:  number:      624          0            0
rcu: cputime:       49          0         2446   ==> 2500(ms)
```

3. 禁用抢占
```bash
rcu:          hardirqs   softirqs   csw/system
rcu:  number:      624         45            0
rcu: cputime:       69          1         2425   ==> 2500(ms)
```

4. 大量的中断导致rcu stall
```bash
rcu:          hardirqs   softirqs   csw/system
rcu:  number:       xx         xx            0
rcu: cputime:       xx         xx            0   ==> 2500(ms)
```

上面的测试驱动加载后，并打开rcu_cpu_stall_cputime=1参数，rcu stall打印出的log，从下面的log可以初步判断是由于中断禁用所导致的rcu stall
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241203153011.png)

#### 4.1.3 使用ftrace调试

使用 ftrace的preemptirqsoff tracer，并设置tracing_max_latency（最大延迟阈值）来跟踪内核的调用栈，查看中断关闭时长，定位具体是哪个模块哪个函数

编译内核时打开CONFIG_FTRACE, CONFIG_FUNCTION_TRACER, CONFIG_IRQSOFF_TRACER, CONFIG_PREEMPT_TRACER

使用的ftrace调试脚本如下：

```bash
#!/bin/bash
# 清空trace
echo > /sys/kernel/debug/tracing/trace
# 设置irqsoff tracer
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
# 设置最大延迟阈值，单位是ns，调试rcu stall问题可以设置大一点，这里设置1s
echo 1000000000 > /sys/kernel/debug/tracing/tracing_max_latency
# 加载测试驱动
echo "insmod rcu_stall_test.ko"
insmod /home/jack/Downloads/rcu_stall_test.ko

echo "start tracing"
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 35

echo 0 > /sys/kernel/debug/tracing/tracing_on
echo "rcu stall test finished"
```

打印trace，可以看到测试驱动中rcu_stall_simulation()中的中断关闭延时约为30s左右，可以定位到这里可能会出现rcu stall的问题
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241203141552.png)

## 5. 参考链接

1. [https://www.cnblogs.com/LoyenWang/p/12681494.html](https://www.cnblogs.com/LoyenWang/p/12681494.html)
2. [https://docs.kernel.org/RCU/stallwarn.html](https://docs.kernel.org/RCU/stallwarn.html)
3. [https://docs.kernel.org/RCU/whatisRCU.html](https://docs.kernel.org/RCU/whatisRCU.html)