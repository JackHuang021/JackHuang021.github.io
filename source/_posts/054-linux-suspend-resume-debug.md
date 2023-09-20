---
title: Linux睡眠唤醒调试方法
tags:
  - Linux
  - suspend
categories: Linux
abbrlink: f183bd8
date: 2023-09-18 16:17:10
---

Linux中的挂起、休眠，一般是指以下四种状态：
1. STI(suspend to idle)：是一种通用的、纯软件、轻量级系统睡眠状态
2. Standby(power-on suspend)：是一种适度的功耗节省状态，同时系统也可以比较快的唤醒
3. STR(suspend to ram)：提供了比较显著的功耗节省，系统中除了内存之外的部件都进入了低功耗状态
4. STD(suspend to disk)：提供最大程度的功耗节省
<!-- more -->

| State | ACPI state | Label |
| :-: | :-: | :-: |
| STI | S0 | s2idle/freeze |
| standby | S1 | shallow/standby |
| STR | S3 | deep |
| STD | S4 | disk |

> 具体的状态描述可以参考内核文档[Documentation/power/states.txt](https://www.kernel.org/doc/Documentation/power/states.txt)

### 基础调试方法
1. 关闭串口睡眠：在启动参数中增加`no_console_suspend ignore_loglevel`，S3调试日志如下
```bash
root@Ubuntu:~# echo mem > /sys/power/state
[   74.403728] PM: suspend entry (deep)
[   74.411597] Filesystems sync: 0.004 seconds
[   74.576746] Freezing user space processes ... (elapsed 0.003 seconds) done.
[   74.586948] OOM killer disabled.
[   74.590196] Freezing remaining freezable tasks ... (elapsed 0.001 seconds) done.
[   74.604967] pcieport 0000:00:04.0: pciehp: Timeout on hotplug command 0x1438 (issued 71912 msec ago)
[   74.605025] pcieport 0000:00:05.0: pciehp: Timeout on hotplug command 0x1438 (issued 71884 msec ago)
[   74.617628] macb 3200c000.ethernet eth0: Link is Down
[   74.629279] macb 3200c000.ethernet: gem-ptp-timer ptp clock unregistered.
[   74.752892] pcieport 0000:00:01.0: pciehp: Timeout on hotplug command 0x1438 (issued 72084 msec ago)
[   76.636865] pcieport 0000:00:04.0: pciehp: Timeout on hotplug command 0x0418 (issued 2020 msec ago)
[   76.644855] pcieport 0000:00:05.0: pciehp: Timeout on hotplug command 0x0418 (issued 2024 msec ago)
[   76.784854] pcieport 0000:00:01.0: pciehp: Timeout on hotplug command 0x0418 (issued 2024 msec ago)
[   76.917236] Disabling non-boot CPUs ...
```

2. 在启动参数中加入`initcall_debug`，打印init函数的进入和返回log，可以定位哪个init函数运行失败或运行时间过长，S3调试日志如下
```bash
root@Ubuntu:~# echo mem > /sys/power/state
[  409.072355] PM: suspend entry (deep)
[  409.087412] Filesystems sync: 0.011 seconds
[  409.260860] Freezing user space processes ... (elapsed 0.003 seconds) done.
[  409.271298] OOM killer disabled.
[  409.274561] Freezing remaining freezable tasks ... (elapsed 0.001 seconds) done.
[  409.284345] rtc rtc0: calling rtc_suspend+0x0/0x138 @ 840, parent: 0-0068
[  409.291569] rtc rtc0: rtc_suspend+0x0/0x138 returned 0 after 317 usecs
[  409.298209] mtd mtd1ro: calling mtd_cls_suspend+0x0/0x78 @ 840, parent: 10000000.lbc_nor
[  409.306323] mtd mtd1ro: mtd_cls_suspend+0x0/0x78 returned 0 after 1 usecs
[  409.313140] mtd mtd1: calling mtd_cls_suspend+0x0/0x78 @ 840, parent: 10000000.lbc_nor
[  409.321075] mtd mtd1: mtd_cls_suspend+0x0/0x78 returned 0 after 1 usecs
...
[  412.886230] Disabling non-boot CPUs ...
```

3. 禁用异步suspend resume设备，排除设备驱动的pm问题`echo 0 > /sys/power/pm_async`，在Linux异步对设备进行suspend resume时出现问题的情况下，可以禁用异步suspend resume来排查设备驱动pm问题

### PM_DEBUG选项调试休眠唤醒
打开CONFIG_PM_DEBUG选项，使用`/sys/power/pm_test`来测试休眠、唤醒，pm_test里面一共有5种测试模式

+ freezer：测试进程冻结
+ devices：测试进程冻结和设备驱动suspend和resume
+ platform：测试进程冻结、设备驱动suspend和resume、suspending platform global control methods
+ processors：测试进程冻结、设备驱动suspend resume、suspending platform global control methods、关闭nonboot CPU
+ core：测试进程冻结、设备驱动suspend resume、关闭nonboot CPU、suspending platform/system devices

使用该方法进行调试时，需要往`/sys/power/pm_test`中写入对应的测试模式，然后进行S3 S4休眠操作，休眠流程走完后，5秒后会自动唤醒系统，测试的时候可以从freezer devices platform...逐步进行测试

> 详细描述可参考内核文档[basic-pm-debugging.txt](https://www.kernel.org/doc/Documentation/power/basic-pm-debugging.txt)

freezer休眠唤醒调试日志
```bash
root@Ubuntu:/sys/power# cat /proc/cmdline 
console=ttyAMA1,115200 audit=0 earlycon=pl011,0x2800d000 root=/dev/nvme0n1p2 rw no_console_suspend initcall_debug ignore_loglevel
root@Ubuntu:/sys/power# echo freezer > pm_test
root@Ubuntu:/sys/power# echo mem > state 
[ 1341.488023] PM: suspend entry (deep)
[ 1341.501723] Filesystems sync: 0.009 seconds
[ 1341.624379] [drm] can not get n_m for link_rate(270000) and sample_rate(0)
[ 1341.779698] Freezing user space processes ... (elapsed 0.003 seconds) done.
[ 1341.790421] OOM killer disabled.
[ 1341.793664] Freezing remaining freezable tasks ... (elapsed 0.001 seconds) done.
[ 1341.802539] PM: suspend debug: Waiting for 5 second(s).
[ 1346.808159] OOM killer enabled.
[ 1346.811304] Restarting tasks ... done.
[ 1346.817354] PM: suspend exit
root@Ubuntu:/sys/power# 
```

devices休眠唤醒调试日志
```bash
root@Ubuntu:/sys/power# cat /proc/cmdline 
console=ttyAMA1,115200 audit=0 earlycon=pl011,0x2800d000 root=/dev/nvme0n1p2 rw no_console_suspend initcall_debug loglevel=7
root@Ubuntu:/sys/power# echo devices > pm_test
root@Ubuntu:/sys/power# echo mem > state 

[ 1628.921417] PM: suspend entry (deep)
[ 1628.934839] Filesystems sync: 0.009 seconds
[ 1629.066342] Freezing user space processes ... (elapsed 0.003 seconds) done.
[ 1629.076995] OOM killer disabled.
[ 1629.080260] Freezing remaining freezable tasks ... (elapsed 0.001 seconds) done.
[ 1629.090074] rtc rtc0: calling rtc_suspend+0x0/0x138 @ 998, parent: 0-0068
[ 1629.097263] rtc rtc0: rtc_suspend+0x0/0x138 returned 0 after 309 usecs
[ 1629.103874] mtd mtd1ro: calling mtd_cls_suspend+0x0/0x78 @ 998, parent: 10000000.lbc_nor
[ 1629.111983] mtd mtd1ro: mtd_cls_suspend+0x0/0x78 returned 0 after 0 usecs
[ 1629.118818] mtd mtd1: calling mtd_cls_suspend+0x0/0x78 @ 998, parent: 10000000.lbc_nor
[ 1629.126754] mtd mtd1: mtd_cls_suspend+0x0/0x78 returned 0 after 2 usecs
...
[ 1632.473232] PM: suspend debug: Waiting for 5 second(s).
[ 1637.479347] reg-dummy reg-dummy: calling platform_pm_resume+0x0/0x60 @ 998, parent: platform
[ 1637.487835] reg-dummy reg-dummy: platform_pm_resume+0x0/0x60 returned 0 after 0 usecs
[ 1637.495697] regulator regulator.0: calling regulator_resume+0x0/0x190 @ 998, parent: reg-dummy
[ 1637.504334] regulator regulator.0: regulator_resume+0x0/0x190 returned 0 after 1 usecs
...
[ 1639.619802] nvme 0000:01:00.0: pci_pm_resume+0x0/0xb8 returned 0 after 12 usecs
[ 1639.628076] OOM killer enabled.
[ 1639.631235] Restarting tasks ... done.
[ 1639.651582] PM: suspend exit
[ 1639.665907] nvme nvme0: Shutdown timeout set to 8 seconds
[ 1639.769959] nvme nvme0: 2/0/0 default/read/poll queues
[ 1641.640767] macb 3200c000.ethernet eth0: yt8521_read_status, phy addr: 0, link up, media: UTP, mii reg 0x11 = 0xbc00
[ 1641.651931] macb 3200c000.ethernet eth0: unable to generate target frequency: 125000000 Hz
[ 1641.661478] macb 3200c000.ethernet eth0: Link is Up - 1Gbps/Full - flow control off
```


### 总结
1. 调试休眠和唤醒首先配置启动参数`no_console_suspend initcall_debug ignore_loglevel`，通过打印的日志信息基本上可以排查出问题原因
2. 通过上面步骤无法排查出阻碍休眠的原因时，可以打开`CONFIG_PM_DEBUG`选项，往`/sys/power/pm_test`中写入freezer devices platform...逐步进行测试，观察哪个测试无法通过，可以对应查找问题


