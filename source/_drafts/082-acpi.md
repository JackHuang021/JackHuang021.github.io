---
title: ACPI 电源管理
tags:
---

## 1. ACPI 介绍

ACPI (Advanced Configuration and Power Interface)，是由 Intel, Microsoft 和 Toshiba 在 1996 年共同提出的一个开放标准，旨在为操作系统提供硬件发现、配置、电源管理和设备控制等功能的统一接口。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250523141400.png)

## 2. 状态定义

### 2.1 全局系统状态定义

全局系统状态（即 Gx 状态）应用到整个系统，是用户可以看见的状态，全局系统状态主要依据如下六个准则划分：
1. 应用软件是否在运行？
2. 发生外部事件后，应用程序响应此事件的延时是多少？
3. 电量消耗是多少？
4. 返回到工作状态，是否需要重启OS？
5. 是否可以拆卸计算机硬件？
6. 是否可以通过外设进入和退出此状态？

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250522141812.png)

+ **G0工作状态**：系统正常执行应用程序，外部设备动态地改变自身的电源状态，用户可以通过用户接口程序选择系统的各种性能、电源状态，系统可以实时地对外部事件进行响应。
+ **G1睡眠状态**：在此状态下计算机不执行用户程序，系统看上去就像关机一样，计算机消耗很少电量，可以通过唤醒进入到工作状态。
+ **G2软关机状态**：在此状态下，计算机已经被关机了，计算机主板仍有少量供电（例如 电源按键电路、网络唤醒、RTC定时唤醒等）
+ **G3机械关机状态**：计算机通过机械式的操作断开电源，此状态下可以安全拆卸计算机硬件

### 2.2 设备电源状态定义

设备电源状态（即 Dx 状态）对用户通常是不可见的，尽管系统作为一个整体仍然处于工作状态，但是有一些设备可能已经处于关闭状态。设备电源状态如下表：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250522145009.png)

### 2.3 睡眠状态定义

睡眠状态（即 Sx 状态）中的 S1~S3 是在全局系统状态 G1 下的睡眠状态类型，5 种睡眠状态的定义如下：

+ **S1 睡眠状态**：低唤醒延时的睡眠状态，在此状态下，硬件维护所有的系统上下文
+ **S2 睡眠状态**：低唤醒延时的睡眠状态，除了未保存 CPU 和系统缓存的上下文外，此状态类似于 S1 睡眠状态。
+ **S3 睡眠状态**：低唤醒延时的睡眠状态，系统上下文保存在内存中，内存保持供电
+ **S4 睡眠状态**：是 ACP 支持的最低电量消耗、最长唤醒延时的睡眠状态，系统上下文保存在硬盘中，计算机完全断电
+ **S5 软关机状态**：不保存任何上下文，计算机完全断电

### 2.4 处理器电源状态定义

处理器电源状态 （即 Cx 状态）是处理器在全局系统状态 G0 下的电量消耗和散热管理状态，为了在工作状态下进一步实现节省电量的目的，OS 在空闲时会将 CPU 置于更低的电量状态，处理器的电源状态定义如下：

+ **C0 状态**：处理器在此状态时正常执行指令
+ **C1 状态**：CPU 不执行指令，但是能快速响应中断
+ **C2 状态**：CPU 停止时钟，响应延迟稍高
+ **C3 状态**：CPU cache 关闭，不再与内存保持一致性，唤醒后可能需要刷新缓存，响应延迟最高

### 2.5 设备和处理器性能状态定义

设备和处理器性能状态（即 Px 状态）是设备或者处理器在执行状态下（处理器在 C0 状态，设备在 D0 状态）的电量消耗和能力状态。性能状态的定义如下：
+ **P0 性能状态**：当设备或者处理器在此状态时，会达到最高的性能，消耗最大的电量
+ **P1 性能状态**：在此状态下，设备或者处理器的性能能力被限制在最高能力之下，设备或者处理器会消耗比最大电量少的电量。
+ **Pn 性能状态**：在此状态下，设备或者处理器的性能能力处于最低程度，消耗维持工作状态所需的最少电量。n是最大的性能状态数，其值依赖于处理器或设备。处理器和设备可以支持的性能状态不能超过16个

### 2.6 全局系统电源状态切换关系图

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250522152720.png)

## 3. ACPI下的设备电源管理

在 ACPI 中，设备的电源管理是其核心功能之一。ACPI通过定义统一的接口，使操作系统可以直接控制设备的电源状态，从而实现精细化的电源管理，降低功耗，提高系统能效。

设备电源状态的切换依赖于ACPI表中定义的`_PSx`（即 Power State）方法
+ _PS0：设备电源状态切换到D0的控制方法
+ _PS1：切换到D1
+ _PS2：切换到D2
+ _PS3：切换到D3

一个支持电源控制的设备在 ACPI 表中的描述大致如下：

```bash
Device (DEV0)
{
    Name (_HID, "PNP0C0A")     // 硬件ID
    Name (_PR0, Package() { ... })   // 进入D0时依赖的电源资源（电源域）
    Name (_PR3, Package() { ... })   // 进入D3时需关闭的电源资源
    Method (_PS0, 0, NotSerialized)  // 切换到D0状态
    {
        // 控制寄存器或GPIO
    }
    Method (_PS3, 0, NotSerialized)  // 切换到D3
    {
        // 断电
    }
}
```

ACPI 设备电源状态切换的调用路径：

```c
// drivers/acpi/power.c
acpi_bus_set_power()
  acpi_device_set_power()
     acpi_power_transition()
        acpi_evaluate_object(..., "_PSx")
```

ACPI 中对设备的电源管理可以与 linux Runtime PM集成，系统调用设备驱动的 suspend/resume 方法时会通过 ACPI 实现设备电源状态的切换

## 4. ACPI下的CPU电源管理

采用 UEFI+ACPI 的启动方式，在 Phytium 的 SoC 上仍是走 PSCI （Power State Coordination Interface） 实现 CPU 的上下电，系统待机休眠，这也是目前 ARM64 平台的主流方案。

PSCI 是由 ARM 定义的标准接口，用于在非特权模式下控制 CPU suspend/resume, hotplug, 电源域管理。

ARM64 平台 不执行 ACPI 控制方法（如 _S3/_S5）来控制电源状态，而是在 `drivers/firmware/psci/psci.c` 中注册了 PSCI 的控制接口，这些接口最终通过 SMC 指令调用进入 EL3 固件完成电源状态切换。

```c
psci_ops = (struct psci_operations){
    .get_version = psci_0_2_get_version,
    .cpu_suspend = psci_0_2_cpu_suspend,
    .cpu_off = psci_0_2_cpu_off,
    .cpu_on = psci_0_2_cpu_on,
    .migrate = psci_0_2_migrate,
    .affinity_info = psci_affinity_info,
    .migrate_info_type = psci_migrate_info_type,
};

static const struct platform_suspend_ops psci_suspend_ops = {
	.valid          = suspend_valid_only_mem,
	.enter          = psci_system_suspend_enter,
};
```

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250523142258.png)

## 5. 参考

1. 《ACPI_Spec_6_5》
2. [http://www.tup.tsinghua.edu.cn/upload/books/yz/064076-01.pdf](http://www.tup.tsinghua.edu.cn/upload/books/yz/064076-01.pdf )  
