---
title: openocd+jlink+gdb 调试linux内核
tags:
---

本文介绍在X86 Ubuntu 22.04宿主机上使用openocd + jlink + gdb搭建一个调试E2000Q Demo开发板Linux 内核的环境。

#### 1. 安装openocd
安装openocd,`sudo apt-get install openocd`

#### 2. 安装jlink驱动
驱动下载地址[https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.deb](https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.deb)，使用dpkg命令安装deb包

#### 3. 内核config修改
1. 为了方便调试，关闭基地址随机化，配置 `CONFIG_RANDOMIZE_BASE=n`
2. 为了得到调试符号，设置 `CONFIG_DEBUG_INFO=y`

#### 4. 配置openocd连接开发板
1. jlink连接开发板JTAG接口，开发板上电启动
![](https://raw.githubusercontent.com/JackHuang021/images/master/微信图片_20240927165032.jpg)

2. 使用黄鹤他们写的openocd配置文件连接开发板，[开发环境工具包下载地址](https://gitee.com/link?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F1J7dndPqMtQD1RNmfdVHblQ)，下载该开发工具包并解压，配置文件位于`phytium-rtos-dev-tools/openocd/share/openocd/scripts/`下，对应的配置文件为`e2000d.cfg, e2000d_cmsisdap_v1.cfg, e2000d_jlink_v9.cfg
`
```bash
sudo openocd \
-f phytium-rtos-dev-tools/openocd/share/openocd/target/e2000d_jlink_v9.cfg \
-s phytium-rtos-dev-tools/openocd/share/openocd/scripts/
```

出现下面的信息即表示连接成功，且gdb server也被成功启动
```bash
jack@linux:~/Downloads/jtag_debug $ openocd -f e2000d_jlink_v9.cfg
Open On-Chip Debugger 0.11.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : Hardware thread awareness created
reboot_e2000_core
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : J-Link V9 compiled Dec 13 2022 11:14:50
Info : Hardware version: 9.20
Info : VTarget = 3.314 V
Info : clock speed 10000 kHz
Info : JTAG tap: e2000.tap tap/device found: 0x5ba00477 (mfg: 0x23b (ARM Ltd), part: 0xba00, ver: 0x5)
Info : DAP transaction stalled (WAIT) - slowing down
Info : DAP transaction stalled (WAIT) - slowing down
Info : e2000.cpu0: hardware has 6 breakpoints, 4 watchpoints
Info : starting gdb server for e2000.cpu0 on 3333
Info : Listening on port 3333 for gdb connections
```

#### 5. 使用GDB调试Linux内核
1. 启动GDB并加载你的内核符号表（vmlinux文件）：
```bash
jack@linux:~/Documents/phytium/source/linux/6_6 (v6.6_dev*) $ gdb-multiarch vmlinux
```

2. 连接到openocd
```bash
(gdb) target remote localhost:3333
```

#### 6. 调试操作
1. 设置断点
```bash
(gdb) break start_kernel
```

2. 继续运行
```bash
(gdb) continue
```

2. 查看寄存器状态
```bash
(gdb) info registers 
x0             0x5                 5
x1             0xffff0020fef88bd8  18446462740449496024
x2             0x3fc939            4180281
x3             0x0                 0
x4             0xffff80207da74000  18446603475768262656
x5             0x4000000000000000  4611686018427387904
x6             0x136375cc9ea       1332368689642
x7             0xffff0020fefa1bc0  18446462740449598400
x8             0xffff0020fefa1c40  18446462740449598528
x9             0x0                 0
x10            0x0                 0
x11            0x32                50
x12            0x0                 0
x13            0x7                 7
x14            0x73276f4aa89       7913325505161
x15            0x6d1a2a98662       7497446950498
x16            0x2                 2
x17            0x20                32
x18            0x0                 0
x19            0x0                 0
x20            0xffff800081739a40  18446603338393033280
x21            0xffff800081739b44  18446603338393033540
x22            0xffff8000817439c0  18446603338393074112
x23            0x0                 0
x24            0x0                 0
x25            0xffff8000817439c0  18446603338393074112
x26            0xf9dc5f38          4191969080
x27            0x0                 0
x28            0x916150ac          2439073964
x29            0xffff800081733d70  18446603338393009520
x30            0xffff800080eb6ad0  18446603338384108240
sp             0xffff800081733d70  0xffff800081733d70
pc             0xffff800080eb6abc  0xffff800080eb6abc <cpu_do_idle+8>
cpsr           0x600000c5          [ SP=1 EL=1 nRW=0 F I C Z ]
fpsr           0x11                17
fpcr           0x0                 0
ELR_EL1        0xffff800080eb7724  0xffff800080eb7724 <default_idle_call+40>
ESR_EL1        0x56000000          1442840576
SPSR_EL1       0x60000005          1610612741
ELR_EL2        0x0                 0x0
ESR_EL2        0x0                 0
SPSR_EL2       0x0                 0
ELR_EL3        0x0                 0x0
ESR_EL3        0x0                 0
SPSR_EL3       0x0                 0
```

4. 停止调试和清理：在GDB中输入quit退出，然后终止OpenOCD进程。