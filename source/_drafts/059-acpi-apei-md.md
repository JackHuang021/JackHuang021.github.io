---
title: 059_acpi_apei.md
date: 2023-11-21 09:24:12
tags:
  - acpi
  - apei
categories: Linux
---

## 1. APEI(ACPI Platform Error Interfaces)

APEI提供了platform固件向OS传递错误信息的途径

APEI由四个分表组成：

+ Error Record Serialization Table(ERST)：用来存储错误
+ Boot Error Record Table(BERT)： 主要用来记录在启动过程中出现的错误
+ Hardware Error Source Table(HEST)：定义硬件错误，标准化硬件错误接口的实现
+ Error Injection Table(EINJ)：主要用来注入错误并触发错误

### 1.1 硬件错误类型

硬件错误可分为可纠错误（Corrected Error）和不可纠错误(Uncorrected Error)

+ 可纠错误： 可由硬件自身或固件进行修正的硬件错误，上报至OS时故障已经被修正。
+ 不可纠错误： 不能由硬件自身或固件进行修正的硬件错误，不可纠错误又分为致命错误(Fatal)或非致命错误（Non-fatal）。当发生致命错误时，系统会重启以防止错误带来的影响；非致命错误发生时，OS会尝试进行修复。

### 1.2 错误源

platform固件会通过描述错误源的表向OS枚举错误源，在初始化期间，OS会检查这些表，并建立必要的错误处理程序，负责处理来自固件的错误通知

## 2. Boot Error Source

Boot Error Source用来上报上次启动过程中未处理的错误，这个错误源描述在BERT表中，Boot Error存放在一段OS可访问的内存中，这段内存被固件保留，Boot Error的存放格式遵循通用硬件错误源结构（Generic Hardware Error Source Structure）中定义的错误状态块的格式，S5000C实现了BERT表，D3000无BERT表。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121102501.png)

### 2.1 S5000C BERT表描述

飞腾S5000C的BERT表如下，boot error source存放地址位于0xFB080000，长度为4kB：

```c
[000h 0000   4]                    Signature : "BERT"    [Boot Error Record Table]
[004h 0004   4]                 Table Length : 00000030
[008h 0008   1]                     Revision : 01
[009h 0009   1]                     Checksum : 15
[00Ah 0010   6]                       Oem ID : "PHYLTD"
[010h 0016   8]                 Oem Table ID : "PHYTIUM."
[018h 0024   4]                 Oem Revision : 00000001
[01Ch 0028   4]              Asl Compiler ID : "PHYT"
[020h 0032   4]        Asl Compiler Revision : 00000001

[024h 0036   4]     Boot Error Region Length : 00001000
[028h 0040   8]    Boot Error Region Address : 00000000FB080000

Raw Table Data: Length 48 (0x30)

    0000: 42 45 52 54 30 00 00 00 01 15 50 48 59 4C 54 44  // BERT0.....PHYLTD
    0010: 50 48 59 54 49 55 4D 2E 01 00 00 00 50 48 59 54  // PHYTIUM.....PHYT
    0020: 01 00 00 00 00 10 00 00 00 00 08 FB 00 00 00 00  // ................
```

## 3. ACPI Error Source

硬件错误源（Hardware Error Source）是一种描述错误源的标准化机制，使用该接口进行硬件错误源描述是固件首选的方式，这种方式独立于处理器架构，HEST（Hardware Error Source Table）提供固件向OS描述硬件错误源的一种方法。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121151010.png)

### 3.1 S5000C和D3000的 HEST表实例

```c
[000h 0000   4]                    Signature : "HEST"    [Hardware Error Source Table]
[004h 0004   4]                 Table Length : 00000250
[008h 0008   1]                     Revision : 01
[009h 0009   1]                     Checksum : EE
[00Ah 0010   6]                       Oem ID : "PHYLTD"
[010h 0016   8]                 Oem Table ID : "PHYTIUM."
[018h 0024   4]                 Oem Revision : 00000001
[01Ch 0028   4]              Asl Compiler ID : "PHYT"
[020h 0032   4]        Asl Compiler Revision : 00000001
// 一共有6个错误源，全部是GHESv2类型的
[024h 0036   4]           Error Source Count : 00000006
```

### 3.2 GHESv2(Generic Hardware Error Source version 2)

固件可以使用GHES来描述一个通用的硬件错误源给OS，OS会配置错误处理机制从error status block（故障信息记录的一段内存）中去读取错误，OS读取错误后需要清除error status block并写Read Ack Register来告诉RAS控制器已经处理过该错误。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121134740.png)

S5000C和D3000上的一个GHESv2实例

```c
// 错误源类型为GHESv2
[028h 0040   2]                Subtable Type : 000A [Generic Hardware Error Source V2]
[02Ah 0042   2]                    Source Id : 0000
[02Ch 0044   2]            Related Source Id : FFFF
[02Eh 0046   1]                     Reserved : 00
[02Fh 0047   1]                      Enabled : 01
[030h 0048   4]       Records To Preallocate : 00000001
[034h 0052   4]      Max Sections Per Record : 00000001
// 错误源记录大小为4kB
[038h 0056   4]          Max Raw Data Length : 00001000
// Generic Address Structure，见ACPI Spec 6.5 Table5.1，用来描述error status block位置
[03Ch 0060  12]         Error Status Address : [Generic Address Structure]
[03Ch 0060   1]                     Space ID : 00 [SystemMemory]
[03Dh 0061   1]                    Bit Width : 40
[03Eh 0062   1]                   Bit Offset : 00
[03Fh 0063   1]         Encoded Access Width : 04 [QWord Access:64]
[040h 0064   8]                      Address : 00000000FB100008
// Hardware Error Notification Structure 见ACPI Spec 6.5 Table 18.14
[048h 0072  28]                       Notify : [Hardware Error Notification Structure]
[048h 0072   1]                  Notify Type : 0B [Software Delegated Exception]
[049h 0073   1]                Notify Length : 5C
[04Ah 0074   2]   Configuration Write Enable : 003E
[04Ch 0076   4]                 PollInterval : 00000014
[050h 0080   4]                       Vector : 00000001
[054h 0084   4]      Polling Threshold Value : 00000000
[058h 0088   4]     Polling Threshold Window : 00000000
[05Ch 0092   4]        Error Threshold Value : 00000000
[060h 0096   4]       Error Threshold Window : 00000000

[064h 0100   4]    Error Status Block Length : 00001000
// Generic Address Structure，见ACPI Spec 6.5 Table5.1，用来描述read ack register位置
[068h 0104  12]            Read Ack Register : [Generic Address Structure]
[068h 0104   1]                     Space ID : 00 [SystemMemory]
[069h 0105   1]                    Bit Width : 40
[06Ah 0106   1]                   Bit Offset : 00
[06Bh 0107   1]         Encoded Access Width : 04 [QWord Access:64]
[06Ch 0108   8]                      Address : 00000000FB100000
// read ack register只用到了bit0
[074h 0116   8]            Read Ack Preserve : FFFFFFFFFFFFFFFE
[07Ch 0124   8]               Read Ack Write : 0000000000000001
```

## 4. 错误注入

OS通过这种机制将硬件错误注入到固件，然后固件上报错误，用来测试OS的错误处理机制是否正常工作。

### 4.1 Error Injection Table

![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121142126.png)

### 4.2 S5000C的EINJ表实例

S5000C的EINJ表实例如下（D3000 的RAS固件没有实现EINJ表）：

```c
[000h 0000   4]                    Signature : "EINJ"    [Error Injection table]
[004h 0004   4]                 Table Length : 00000170
[008h 0008   1]                     Revision : 01
[009h 0009   1]                     Checksum : DE
[00Ah 0010   6]                       Oem ID : "PHYLTD"
[010h 0016   8]                 Oem Table ID : "PHYTIUM."
[018h 0024   4]                 Oem Revision : 00000001
[01Ch 0028   4]              Asl Compiler ID : "PHYT"
[020h 0032   4]        Asl Compiler Revision : 00000001

[024h 0036   4]      Injection Header Length : 00000030
[028h 0040   1]                        Flags : 00
[029h 0041   3]                     Reserved : 000000
// 一共有10个注入操作表
[02Ch 0044   4]        Injection Entry Count : 0000000A
```

### 4.3 注入指令描述

一个注入操作通过一条或多条注入指令组成，一条注入指令及对寄存器的一个操作，一个注入指令入口描述一个注入硬件寄存器，和该指令对这个寄存器的操作

```c
// 一共有12种注入操作，见 ACPI Spec 6.5 Table 18.25
[030h 0048   1]                       Action : 00 [Begin Operation]
// 对寄存器的操作，见ACPI Spec 6.5 Table 18.28，这里是将Vlaue写入到寄存器中
[031h 0049   1]                  Instruction : 03 [Write Register Value]
[032h 0050   1]        Flags (decoded below) : 01
                      Preserve Register Bits : 1
[033h 0051   1]                     Reserved : 00

[034h 0052  12]              Register Region : [Generic Address Structure]
[034h 0052   1]                     Space ID : 00 [SystemMemory]
[035h 0053   1]                    Bit Width : 40
[036h 0054   1]                   Bit Offset : 00
[037h 0055   1]         Encoded Access Width : 04 [QWord Access:64]
// 寄存器地址
[038h 0056   8]                      Address : 00000000FB000000
// 寄存器的值，这里是将该Value写入到寄存器
[040h 0064   8]                        Value : 000000000000FFFF
[048h 0072   8]                         Mask : 00000000FFFFFFFF
```

### 4.4 注入错误类型

![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121144614.png)

### 4.5 错误注入过程

1. 执行BEGIN_INJECTION_OPERATION来通知固件错误注入操作开始了
2. 执行GET_ERROR_TYPE获取固件支持的错误注入类型
3. 执行SET_ERROR_TYPE，设置当前错误注入类型
4. 执行EXECUTE_OPERATION告知固件开始注入
5. 持续执行CHECK_BUSY_STATUS操作，等待固件清除busy bit
6. 执行GET_COMMAND_STATUS，返回当前操作的状态，决定是否进行注入
7. 执行GET_TRIGGER_ERROR_ACTION_TABLE，返回TRIGGER_ERROR action table
8. 执行返回的TRIGGER_ERROR action table中的操作
9. 执行END_OPERATION通知固件错误注入已完成

错误注入过程的代码在`drivers/acpi/apei/einj.c __einj_error_inject()`

## 5. RAS APEI规范测试

### 5.1 内核配置

1. 使能SDEI(Software Delegated Exception Interface)，由S5000C HEST表可知，固件使用SDEI上报RAS错误，需要打开内核**CONFIG_ARM_SDE_INTERFACE**选项
2. 使能FTRACE，rasdaemon收集错误发生时的ftrace事件，然后记录到sqlite3数据库中，打开内核选项**CONFIG_FTRACE**
3. 使能einj模块用于错误注入测试，需要打开内核**CONFIG_ACPI_APEI_EINJ**选项

### 5.2 S5000C 错误注入测试

错误注入接口位于`/sys/kernel/debug/apei/einj/`

查看固件支持的错误注入类型

```bash
root@Ubuntu:/sys/kernel/debug/apei/einj# cat available_error_type 
0x00000001      Processor Correctable
0x00000002      Processor Uncorrectable non-fatal
0x00000004      Processor Uncorrectable fatal
0x00000008      Memory Correctable
0x00000010      Memory Uncorrectable non-fatal
0x00000020      Memory Uncorrectable fatal
0x00000040      PCI Express Correctable
0x00000080      PCI Express Uncorrectable non-fatal
0x00000100      PCI Express Uncorrectable fatal
0x00000200      Platform Correctable
0x00000400      Platform Uncorrectable non-fatal
0x00000800      Platform Uncorrectable fatal
```

Memory Correctable错误注入

```bash
# 设置错误类型
root@Ubuntu:/sys/kernel/debug/apei/einj# echo 0x8 > error_type 
# 设置出错物理内存地址
root@Ubuntu:/sys/kernel/debug/apei/einj# echo 0x80000000 > param1
# 设置内存地址掩码
root@Ubuntu:/sys/kernel/debug/apei/einj# echo 0xfffffffffffff000 > param2
root@Ubuntu:/sys/kernel/debug/apei/einj# echo 0x1 > error_inject 
sdei_ev:0x3     err_src:0x2
-bash: echo: write error: Invalid argument
root@Ubuntu:/sys/kernel/debug/apei/einj# dmesg 
[   99.240202] [Firmware Bug]: APEI: Invalid physical address in GAR [0x0/32/0/3/0]
[   99.240616] {1}[Hardware Error]: Hardware error from APEI Generic Hardware Error Source: 2
[   99.240708] {1}[Hardware Error]: It has been corrected by h/w and requires no further action
[   99.240756] {1}[Hardware Error]: event severity: corrected
[   99.240807] {1}[Hardware Error]:  precise tstamp: 2023-11-21 07:29:39
[   99.240897] {1}[Hardware Error]:  Error 0, type: corrected
[   99.240958] {1}[Hardware Error]:   section_type: ARM processor error
[   99.240991] {1}[Hardware Error]:   MIDR: 0x00000000700f8620
[   99.241043] {1}[Hardware Error]:   Multiprocessor Affinity Register (MPIDR): 0x0000000081000000
[   99.241095] {1}[Hardware Error]:   Error info structure 0:
[   99.241132] {1}[Hardware Error]:   num errors: 1
[   99.241167] {1}[Hardware Error]:    error_type: 0, cache error
[   99.241213] {1}[Hardware Error]:   Context info structure 0:
[   99.241248] {1}[Hardware Error]:    register context type: AArch64 EL1 context registers
[   99.241318] {1}[Hardware Error]:    00000000: 00000000 00000000 00000000 00000000
[   99.241389] {1}[Hardware Error]:    00000010: 00000000 00000000 00000000 00000000
[   99.241459] {1}[Hardware Error]:    00000020: 0044ffff 00000004 700f8620 00000000
[   99.241517] {1}[Hardware Error]:    00000030: 81000000 00000000 30500800 00000000
[   99.241575] {1}[Hardware Error]:    00000040: 0a513ea0 ffff8000 0a513d50 ffff8000
[   99.241630] {1}[Hardware Error]:    00000050: 00000000 00000000 b5503510 000000f5
[   99.241684] {1}[Hardware Error]:    00000060: ffaa0000 00000000 f4a0a000 00000000
[   99.241737] {1}[Hardware Error]:    00000070: f5777000 00000000 00000018 00000000
[   99.241775] {1}[Hardware Error]:   Vendor specific error info has 48 bytes:
[   99.241840] {1}[Hardware Error]:    00000000: 00000000 00000000 00000000 00000000  ................
[   99.241903] {1}[Hardware Error]:    00000010: 00000000 00000000 00000000 00000000  ................
[   99.241962] {1}[Hardware Error]:    00000020: 00000000 00000000 00000000 00000000  ................
[  838.468750] [Firmware Bug]: APEI: Invalid physical address in GAR [0x0/32/0/3/0]
[  838.469270] EDAC MC0: 1 CE Single-bit ECC on BANK 0 DIMM 1 (node:0 card:1 module:0 bank:0 row:8192 col:0 bit_pos:5 DIMM location:BANK 0 DIMM 1 page:0x80000 offset:0x0 grain:4096 syndrome:0x0 - APEI location: n)
[  838.469500] {2}[Hardware Error]: Hardware error from APEI Generic Hardware Error Source: 2
[  838.469555] {2}[Hardware Error]: It has been corrected by h/w and requires no further action
[  838.469600] {2}[Hardware Error]: event severity: corrected
[  838.469647] {2}[Hardware Error]:  precise tstamp: 2023-11-21 07:41:29
[  838.469740] {2}[Hardware Error]:  Error 0, type: corrected
[  838.469792] {2}[Hardware Error]:   section_type: memory error
[  838.469825] {2}[Hardware Error]:   physical_address: 0x0000000080000000
[  838.469874] {2}[Hardware Error]:   physical_address_mask: 0xfffffffffffff000
[  838.469988] {2}[Hardware Error]:   node: 0 card: 1 module: 0 bank: 0 device: 1 row: 8192 column: 0 bit_position: 5 requestor_id: 0x0000000000000001 responder_id: 0x0000000000000001 target_id: 0x000000000000000 
[  838.470044] {2}[Hardware Error]:   error_type: 2, single-bit ECC
[  838.470101] {2}[Hardware Error]:   DIMM location: BANK 0 DIMM 1 
```

### 5.3 D3000 错误注入测试

由于D3000 RAS UEFI固件没有实现EINJ表，所以只能通过D3000的错误注入寄存器来测试HEST功能，手动注入ras_soc的0号错误：

```bash
root@Ubuntu:~# devmem2 0x36C2207C w 0
```

查看内核的打印信息，手动进行错误注入后，可以看到错误信息的打印：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241106163725.png)

### 5.4 rasdaemon收集错误

重新开启一个终端，ssh登入到S5000C主板，打开rasdaemon，并在前端收集错误，在另一个终端注入Memory Correctable错误，可以看到rasdaemon有错误记录打印

```bash
root@Ubuntu:~# rasdaemon -f
rasdaemon: Listening to events for cpus 0 to 15
           <...>-761   [002] d.h1.     0.000105 mc_event 2023-11-21 15:48:29 +0800 1 Corrected error: Single-bit ECC on BANK 0 DIMM 1 (mc: 0 location: 1 address: 0x80000000 grain: 12 APEI location: node:0 card:1 module:0 bank:0 row:8192 col:0 bit_pos:5 DIMM location:BANK 0 DIMM 1 requestorID: 0x0000000000000001 responderID: 0x0000000000000001 targetID: 0x0000000000000001)
```

## 6. 飞腾UPEI

飞腾UPEI介绍：飞腾UPEI是通过uboot固件实现SDEI，固件通过SDEI和OS交互来处理错误事件。

### D3000 飞腾UPEI设备树节点

```c
diff --git a/arch/arm64/boot/dts/phytium/pd2308-demo-a.dts b/arch/arm64/boot/dts/phytium/pd2308-demo-a.dts
index b9c66f1ccfc3..1014ad676065 100644
--- a/arch/arm64/boot/dts/phytium/pd2308-demo-a.dts
+++ b/arch/arm64/boot/dts/phytium/pd2308-demo-a.dts
@@ -7,6 +7,7 @@
 
 /dts-v1/;
 /memreserve/ 0x80000000 0x10000;
+/memreserve/ 0xfa000000 0x02000000;
 
 #include "pd2308.dtsi"
 
@@ -22,9 +23,10 @@ chosen {
 
 	memory@80000000 {
 		device_type = "memory";
-		reg = <0x00 0x80000000 0x00 0x7b000000>;
+		reg = <0x00 0x80000000 0x00 0x7C000000>;
 		numa-node-id = <0x00>;
 	};
+
 };
 
 &uart0 {
diff --git a/arch/arm64/boot/dts/phytium/pd2308.dtsi b/arch/arm64/boot/dts/phytium/pd2308.dtsi
index 763c04ac5d30..cbbc1f1e46b7 100644
--- a/arch/arm64/boot/dts/phytium/pd2308.dtsi
+++ b/arch/arm64/boot/dts/phytium/pd2308.dtsi
@@ -116,6 +116,22 @@ scmi_sensors0: protocol@15 {
 				#thermal-sensor-cells = <1>;
 			};
 		};
+
+		sdei {
+			compatible = "arm,sdei";
+			method = "smc";
+			address = <0xfa000000>;
+		};
+
+		hest {
+			compatible = "hest-1.0";
+			address = <0xfa080000>;
+		};
+
+		einj {
+			compatible = "einj-1.0";
+			address = <0xfa280000>;
+		};
 	};
 
 	thermal-zones {
```

### S5000C 飞腾UPEI设备树节点

```bash
diff --git a/arch/arm64/boot/dts/phytium/ps2316-devboard-16c-dsk.dts b/arch/arm64/boot/dts/phytium/ps2316-devboard-16c-dsk.dts
index df7194291207..e114d73f8100 100644
--- a/arch/arm64/boot/dts/phytium/ps2316-devboard-16c-dsk.dts
+++ b/arch/arm64/boot/dts/phytium/ps2316-devboard-16c-dsk.dts
@@ -7,6 +7,7 @@
 
 /dts-v1/;
 /memreserve/ 0x80000000 0x10000;
+/memreserve/ 0xfa000000 0x02000000;
 
 #include "ps2316-generic-psci-soc.dtsi"
 
diff --git a/arch/arm64/boot/dts/phytium/ps2316-generic-psci-soc.dtsi b/arch/arm64/boot/dts/phytium/ps2316-generic-psci-soc.dtsi
index b2966e9f6751..4166944c2328 100644
--- a/arch/arm64/boot/dts/phytium/ps2316-generic-psci-soc.dtsi
+++ b/arch/arm64/boot/dts/phytium/ps2316-generic-psci-soc.dtsi
@@ -299,6 +299,21 @@ scmi_sensors0: protocol@15 {
                                #thermal-sensor-cells = <0x01>;
                        };
                };
+
+
+               sdei {
+                       compatible = "arm,sdei";
+                       method = "smc";
+                       address = <0xfa000000>;
+               };
+               hest {
+                       compatible = "hest-1.0";
+                       address = <0xfa080000>;
+               };
+               einj {
+                       compatible = "einj-1.0";
+                       address = <0xfa280000>;
+               };
        };
```

UPEI相关的内核config:

+ CONFIG_PHYT_UPEI
+ CONFIG_PHYT_UPEI_GHES
+ CONFIG_PHYT_UPEI_PCIEAER
+ CONFIG_PHYT_UPEI_SEI
+ CONFIG_PHYT_UPEI_MEMORY_FAILURE
+ CONFIG_PHYT_UPEI_EINJ
+ CONFIG_PHYT_UPEI_ERST_DEBUG

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250121162231.png)

另外还需要打开SDEI驱动

+ CONFIG_ARM_SDE_INTERFACE
+ CONFIG_ARM_SDE_INTERFACE_OF

### 6.1 HEST硬件错误初始化流程

1. 从共享内存地址`0xfa080000`读取HEST表，该地址后面存放的是`error_source_count`个GHES类型的错误描述

    ```c
    struct phyt_table_header {
        char signature[PHYT_NAMESEG_SIZE];	/* ASCII table signature */
        u32 length;		/* Length of table in bytes, including this header */
        u8 revision;		/* PHYT Specification minor version number */
        u8 checksum;		/* To make sum of entire table == 0 */
        char oem_id[PHYT_OEM_ID_SIZE];	/* ASCII OEM identification */
        char oem_table_id[PHYT_OEM_TABLE_ID_SIZE];	/* ASCII OEM table identification */
        u32 oem_revision;	/* OEM revision number */
        char asl_compiler_id[PHYT_NAMESEG_SIZE];	/* ASCII ASL compiler vendor ID */
        u32 asl_compiler_revision;	/* ASL compiler version */
    };

    struct phyt_table_hest {
        struct phyt_table_header header;	/* Common PHYT table header */
        u32 error_source_count;
    };
    ```

2. 根据错误类型解析错误描述`upei_hest_parse(hest_parse_ghes_count, &ghes_count);`
3. 对每一个GHES错误注册ghes platform设备`hest_parse_ghes()`，触发`GHES_OF`驱动注册，调用`ghes_probe()`
4. 调用`sdei_of_register_ghes(ghes, ghes_sdei_normal_callback, ghes_sdei_critical_callback)`注册SDEI事件

### 6.2 SDEI事件处理流程

1. SDEI事件触发，OS调用`__sdei_handler()`来处理这些SDEI事件，接着调用到`do_sdei_event() -> sdei_event_handler()`
2. 调用SDEI事件处理回调函数`arg->callback(event_num, regs, arg->callback_arg);`，这里会调用到GHES错误注册中提供的回调接口
3. GHES错误的回调接口中会触发irq_work，即`ghes_proc_in_irq()`，在`ghes_proc_in_irq()`中遍历所有GHES错误的error status，来判断是哪个错误触发了，然后对错误信息进行打印

### 6.3 D3000 UPEI 错误注入测试

使用`5.3`的方法进行错误注入测试，可以观察到内核的GHES错误信息打印：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250121163403.png)

### 6.4 S5000C UPEI 注入测试

S5000C 固件实现了EINJ表，可以直接写入错误类型进行错误注入

```bash
root@ubuntu:/sys/kernel/debug/apei/einj# echo 0x1 > notrigger 
root@ubuntu:/sys/kernel/debug/apei/einj# echo 0x1 > error_type 
root@ubuntu:/sys/kernel/debug/apei/einj# echo 0x7 > /proc/sys/kernel/printk
root@ubuntu:/sys/kernel/debug/apei/einj# echo 0x1 > error_inject 
pbf printf: ras[0]: sdei_ev:0x3	err_src:0x1
VERBOSE: SDEI:  event state 0x3 => 0x7
VERBOSE: EHF: activate prio=6
VERBOSE: SDEI: > CTX(p:0):81000000
VERBOSE: SDEI: < CTX:-140737179304824
VERBOSE: SDEI: > CTX(p:1):81000000
VERBOSE: SDEI: < CTX:-281472927268624
VERBOSE: SDEI: > CTX(p:2):81000000
VERBOSE: SDEI: < CTX:-140737213917560
VERBOSE: SDEI: > CTX(p:3):81000000
VERBOSE: SDEI: < CTX:1
VERBOSE: SDEI: > COMPLETE(r:1 sta/ep:ffff800010018a80):81000000
VERBOSE: SDEI:  event state 0x7 => 0x3
VERBOSE: SDEI: EOI:81000000, 3 spsr:600003c9 elr:ffff80001001c6fc
VERBOSE: EHF: deactivate prio=-1
[ 1905.169341] {1}[Hardware Error]: Hardware error from APEI Generic Hardware Error Source: 2
[ 1905.400858] {1}[Hardware Error]: It has been corrected by h/w and requires no further action
[ 1905.576336] {1}[Hardware Error]: event severity: corrected
[ 1905.690328] estatus->len:280
[ 1905.750070] {1}[Hardware Error]:  precise tstamp: 2066-01-01 00:02:25
[ 1905.883976] {1}[Hardware Error]:  Error 0, type: corrected
[ 1905.997975] {1}[Hardware Error]:   section_type: ARM processor error
[ 1906.130046] gdata->error_data_length:208
[ 1906.211480] proc->section_length:256
[ 1906.285674] {1}[Hardware Error]:   MIDR: 0x00000000700f8620
[ 1906.401478] {1}[Hardware Error]:   Multiprocessor Affinity Register (MPIDR): 0x0000000081000000
[ 1906.582379] err_info->length:32
[ 1906.647538] {1}[Hardware Error]:   Error info structure 0:
[ 1906.761513] {1}[Hardware Error]:   num errors: 1
[ 1906.857417] {1}[Hardware Error]:    error_type: 0, cache error
[ 1906.978641] ctx_info->size:128
[ 1907.041988] {1}[Hardware Error]:   Context info structure 0:
[ 1907.159590] {1}[Hardware Error]:    register context type: AArch64 EL1 context registers
[ 1907.327862] {1}[Hardware Error]:    00000000: 00000000 00000000 00000000 00000000
[ 1907.483456] {1}[Hardware Error]:    00000010: 00000000 00000000 00000000 00000000
[ 1907.639043] {1}[Hardware Error]:    00000020: 0044ffff 00000004 700f8620 00000000
[ 1907.794626] {1}[Hardware Error]:    00000030: 81000000 00000000 30500800 00000000
[ 1907.950208] {1}[Hardware Error]:    00000040: 117abea0 ffff8000 126bbbd0 ffff8000
[ 1908.105792] {1}[Hardware Error]:    00000050: 00000000 00000000 b5503510 00000035
[ 1908.261380] {1}[Hardware Error]:    00000060: ffaa3000 00000000 812f1000 00000000
[ 1908.416963] {1}[Hardware Error]:    00000070: 81a99000 00000000 00000018 00000000
[ 1908.572540] {1}[Hardware Error]:   Vendor specific error info has 48 bytes:
[ 1908.717272] {1}[Hardware Error]:    00000000: 00000000 00000000 00000000 00000000  ................
[ 1908.905407] {1}[Hardware Error]:    00000010: 00000000 00000000 00000000 00000000  ................
[ 1909.093538] {1}[Hardware Error]:    00000020: 00000000 00000000 00000000 00000000  ................
```

## 7. 总结

S5000C的UEFI固件的RAS功能做的比较完善，而D3000目前给出的RAS固件仅实现了HEST表，无法通过apei的错误注入模块进行错误注入测试，D3000同样有支持RAS的uboot固件，从D3000的软件编程手册来看，应该可以使用phytium edac驱动实现RAS功能。
