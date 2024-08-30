---
title: 059_acpi_apei.md
date: 2023-11-21 09:24:12
tags:
  - acpi
  - apei
categories: Linux
---

### APEI(ACPI Platform Error Interfaces)
APEI提供了platform固件向OS传递错误信息的途径

APEI由四个分表组成：
+ Error Record Serialization Table(ERST)
+ Boot Error Record Table(BERT)
+ Hardware Error Source Table(HEST)
+ Error Injection Table(EINJ)

### 硬件错误类型
硬件错误可分为可纠错误（Corrected Error）和不可纠错误(Uncorrected Error)
+ 可纠错误： 可由硬件自身或固件进行修正的硬件错误，上报至OS时故障已经被修正。
+ 不可纠错误： 不能由硬件自身或固件进行修正的硬件错误，不可纠错误又分为致命错误(Fatal)或非致命错误（Non-fatal）。当发生致命错误时，系统会重启以防止错误带来的影响；非致命错误发生时，OS会尝试进行修复。

### 错误源
platform固件会通过描述错误源的表向OS枚举错误源，在初始化期间，OS会检查这些表，并建立必要的错误处理程序，负责处理来自固件的错误通知

#### Boot Error Source
Boot Error Source用来上报上次启动过程中未处理的错误，这个错误源描述在BERT表中，Boot Error存放在一段OS可访问的内存中，这段内存被固件保留，Boot Error的存放格式遵循通用硬件错误源结构（Generic Hardware Error Source Structure）中定义的错误状态块的格式
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121102501.png)

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

#### ACPI Error Source
硬件错误源（Hardware Error Source）是一种描述错误源的标准化机制，使用该接口进行硬件错误源描述是固件首选的方式，这种方式独立于处理器架构，HEST（Hardware Error Source Table）提供固件向OS描述硬件错误源的一种方法。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121151010.png)

飞腾S5000C HEST表实例
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

##### GHESv2(Generic Hardware Error Source version 2)
固件可以使用GHES来描述一个通用的硬件错误源给OS，OS会配置错误处理机制从error status block（故障信息记录的一段内存）中去读取错误，OS读取错误后需要清除error status block并写Read Ack Register来告诉RAS控制器已经处理过该错误。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121134740.png)

##### 错误通知类型
飞腾S5000C上使用的Software Delegated Exception这种通知类型
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121135247.png)

#### 飞腾S5000C上的一个GHESv2实例
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

### 错误注入
OS通过这种机制将硬件错误注入到固件，然后固件上报错误，用来测试OS的错误处理机制是否正常工作。

#### Error Injection Table
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121142126.png)

飞腾S5000C的EINJ表实例
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

#### 注入指令描述
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

#### 注入错误类型
![](https://raw.githubusercontent.com/JackHuang021/images/master/20231121144614.png)

#### 错误注入过程
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

### S5000C上使用APEI进行错误上报

#### 内核配置
+ 使能SDEI，由S5000C HEST表可知，固件使用SDEI上报RAS错误，需要打开内核CONFIG_ARM_SDE_INTERFACE选项
+ 使能FTRACE，rasdaemon收集错误发生时的ftrace事件，然后记录到sqlite3数据库中，打开内核选项CONFIG_FTRACE

#### 错误注入测试
错误注入接口位于`/sys/kernel/debug/apei/einj/`

##### 查看固件支持的错误注入类型
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

##### Memory Correctable错误注入
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

##### rasdaemon收集错误
重新开启一个终端，ssh登入到S5000C主板，打开rasdaemon，并在前端收集错误，在另一个终端注入Memory Correctable错误，可以看到rasdaemon有错误记录打印
```bash
root@Ubuntu:~# rasdaemon -f
rasdaemon: Listening to events for cpus 0 to 15
           <...>-761   [002] d.h1.     0.000105 mc_event 2023-11-21 15:48:29 +0800 1 Corrected error: Single-bit ECC on BANK 0 DIMM 1 (mc: 0 location: 1 address: 0x80000000 grain: 12 APEI location: node:0 card:1 module:0 bank:0 row:8192 col:0 bit_pos:5 DIMM location:BANK 0 DIMM 1 requestorID: 0x0000000000000001 responderID: 0x0000000000000001 targetID: 0x0000000000000001)
```

