---
title: 087_arm64_cache_ras
tags:
---

## 寄存器介绍

+ CPUMERRSR_EL1: 用于记录 CPU 内核内部（L1 Cache、TLB等）发生的内存错误。

	![](https://raw.githubusercontent.com/JackHuang021/images/master/20250905095004.png)

+ L2MERRSR_EL1: 用于记录 共享 L2 Cache 层级 发生的内存错误。
	![](https://raw.githubusercontent.com/JackHuang021/images/master/20250905095050.png)

Cortex A53、Cortex A57、Cortex A72 中均包含这两个寄存器，这些核属于 ARMv8.0 ~ ARMv8.1 世代，在那个时期 ARM 还没有统一的 RAS Extension 标准（后来在 ARMv8.2 才引入）。从 ARMv8.2 开始，架构引入了 标准化的 RAS Extension，定义了一整套新的、架构规定的错误报告寄存器，因此，从 A55、A76 这种 基于 ARMv8.2+ 的核心开始，就不再使用旧的 CPUMERRSR_EL1 / L2MERRSR_EL1，而是直接实现 RAS 标准寄存器组（在IP核手册Error System registers章节有详细描述）。

## 测试

编写测试驱动对CPUMERRSR_EL1和L2MERRSR_EL1寄存器进行读测试：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250905104707.png)

+ E2000： 无法读取
	![](https://raw.githubusercontent.com/JackHuang021/images/master/20250905103411.png)
+ D3000M： 无法读取
+ 树莓派4（A72）: 可以正常读取
	![](https://raw.githubusercontent.com/JackHuang021/images/master/2025-09-05_09-17.png)

## 总结

飞腾E2000后的处理器都是基于 ARMv8.2 的核心，不适用于a72 edac驱动
