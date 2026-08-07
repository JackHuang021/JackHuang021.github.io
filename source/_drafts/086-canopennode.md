---
title: 086_canopennode
tags:
---

## 1. CANOpenNode介绍

CANopenNode 是一个开源的 CANopen 协议栈实现，用 C 语言编写，目标是为嵌入式设备和 PC 上的 CANopen 应用提供一个轻量、可移植、可扩展的解决方案。它遵循 CiA 301（CANopen 应用层和通信协议） 标准，并且支持常见的 CANopen 设备功能。

CANopenLinux 是在 Linux 系统 上基于 CANopenNode 和 SocketCAN 的一个CANopen实现，它把 CANopen 协议栈封装成一个 Linux 用户态应用，方便在 PC 或嵌入式 Linux 环境下调试、测试、甚至直接部署 CANopen 设备功能。

CANopenNode 项目地址：[https://github.com/CANopenNode/CANopenNode](https://github.com/CANopenNode/CANopenNode)

CANopenLinux 项目地址：[https://github.com/CANopenNode/CANopenLinux](https://github.com/CANopenNode/CANopenLinux)

官方文档地址：[https://canopennode.github.io/](https://canopennode.github.io/)

## 2. 部署CANopenLinux

1. 克隆 CANopenLinux 项目代码，并更新子模块 CANopenNode 的代码

	```bash
	git clone https://github.com/CANopenNode/CANopenLinux.git
	cd CANopenLinux
	git submodule init
	git submodule update
	```

2. 编译 canopend，通过 canopend 可以将 Linux 主机变成 canopen 节点设备

	```bash
	cd CANopenLinux
	make
	sudo make install
	```

3. 编译 cocomm，cocomm 是配套 canopend 使用的命令行交互工具，主要用于在 Linux 下通过 SDO, NMT, LSS 来配置 CANopen 节点，方便调试和测试。可以理解为它是一个轻量级的 CANopen 控制台客户端。

	```bash
	cd CANopenLinux/cocomm
	make
	sudo make install
	```

4. 下载 CANopenEditor 应用程序，CANopenEditor 是 CANopenNode 项目配套的一个图形化对象字典（Object Dictionary）编辑工具，主要用来方便地创建、查看和修改 CANopen 设备的对象字典，并且可以直接生成 CANopenNode 协议栈需要的 CO_OD.c / CO_OD.h 文件。CANopenEditor 的仓库地址为：[https://github.com/CANopenNode/CANopenEditor](https://github.com/CANopenNode/CANopenEditor)，提供 Windows 下的执行程序，目前最新的版本下载地址为： [CANopenEditor-v4.2.3-binary.zip](https://github.com/CANopenNode/CANopenEditor/releases/download/v4.2.3/CANopenEditor-v4.2.3-binary.zip)。

## 3. 测试CANopenNode功能

### 3.1 测试环境搭建

1. 克隆 CANopenDemo 的项目代码，并更新子模块的代码，CANopenDemo 是 CANopenNode 项目里提供的一个示例应用程序，它的主要目的是让开发者快速理解 CANopenNode 协议栈在真实环境中的初始化、运行流程，以及如何和对象字典（OD）及应用层逻辑结合。

	```bash
	git clone https://github.com/CANopenNode/CANopenDemo.git
	cd CANopenDemo
	git submodule update --init --recursive
	```

2. 编译 CANopen demoDevice，CANopen demoDevice 可以生成一个标准的 CANopen 节点，它包含对象字典，其中包含最常见的通信参数以及一些额外的制造商特定参数和设备配置文件参数。可以使用 CANOpenEditor 去编辑 CANopen demoDevice 的对象字典，默认的对象字典参见 `demo/demoDevice.md`。

	```bash
    cd CANopenDemo/demo
    make
	```

3. 硬件接线：将 E2000Q demo 开发板的 CAN0 和 CAN1 连接，将 CAN0、CAN1 连接在一个 CAN 网络上。
	![](https://raw.githubusercontent.com/JackHuang021/images/master/20250808160149.png)

	![](https://raw.githubusercontent.com/JackHuang021/images/master/fa5a2fdd7557de23effc3c0e14be54d7.jpg)

4. 设置 CAN0 和 CAN1 的波特率，并打开 CAN0 CAN1

	```bash
	ip link set can0 type can bitrate 500000 fd off
	ip link set can1 type can bitrate 500000 fd off
	ip link set can0 up
	ip link set can1 up
	```

5. 单独开一个终端，使用 candump 监视 CAN 网络数据

	```bash
	candump -td can0
	```

6. 运行测试程序：在 CAN0 上跑 canopend，生成一个 CANopen节点，其节点 ID 为 1；在 CAN1 上跑 demoLinuxDevice，生成另一个 CANopen 节点，其节点默认 ID 为 4。

	```bash
	canopend can0 -i 1 -c "local-/tmp/CO_command_socket"
	demoLinuxDevice can1
	```

7. 检查环境是否搭建完成：当 canopend 和 demoLinuxDevice 运行起来后，两个 CANopen 节点会发送上线报文，即 0x701 和 0x704 CAN ID 的报文，candump 中能看到上线报文即表示环境搭建没问题。另外还有两条紧急报文 0x81 和 0x84，这是由于两个 CANopen 节点的非易失性存储未初始化导致，只需要对对象字典中的 0x1011 索引中的子索引1 (Restore all default parameters| UNSIGNED32) 写入 load 字符串，下次节点上线便不会再发送紧急报文。

	![](https://raw.githubusercontent.com/JackHuang021/images/master/上线报文.png)

### 3.2 SDO测试

SDO 是 CANopen 协议中用于访问节点对象字典（Object Dictionary）数据的服务。它实现了主站和从站之间的点对点通信，支持读取和写入节点内部参数。

#### 3.2.1 SDO 读取参数测试：读取心跳报文发送参数

查看节点4的心跳报文发送周期，OD索引为0x1017，子索引为0，数据长度为 16 bits，单位为ms。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250808164152.png)

使用 cocomm 与节点 1 交互，发送 SDO 报文

```bash
cocomm "4 read 0x1017 0 u16"
```

SDO 读取过程中的 CAN 报文数据：

![](https://raw.githubusercontent.com/JackHuang021/images/master/获取心跳报文设置.png)

上面的 CAN 报文中，0x604 报文是节点 1 发送的 SDO 请求报文，0x584 报文是节点 4 发送的SDO应答报文，对应的数据为 0，即当前节点 4 不发送心跳报文。

#### 3.2.2 SDO 写入参数测试：设置心跳报文发送参数

使用 cocomm 与节点 1 交互，发送 SDO 报文，修改节点 4 的心跳发送周期为1S

```bash
cocomm "1 write 0x1017 0 u16 1000"
```

CAN 报文数据如下，0x701 报文即为节点 1 的心跳报文，按照周期 1S 发送：

![](https://raw.githubusercontent.com/JackHuang021/images/master/node1心跳数据.png)

### 3.3 NMT测试

NMT 是 CANopen 协议的核心管理机制之一，负责管理网络上所有节点的状态切换和生命周期控制。

#### 3.3.1 通讯复位测试

对节点 4 发送通讯复位指令：

```bash
cocomm "4 reset communication"
```

报文数据如下，0x0 ID的报文即为NMT报文，复位节点4 的通讯（即CAMOpen协议栈重启），可以看到节点4 复位后又重新发送了上线报文 0x704。

![](https://raw.githubusercontent.com/JackHuang021/images/master/通讯复位报文.png)

### 3.4 LSS测试

LSS是 CiA DSP 305 中描述的 CANopen 的扩展。该接口在 CiA DS 309 3.0.0（ASCII 映射）中进行了描述。LSS 允许用户更改节点 ID 和比特率，以及在未配置的节点上设置节点 ID。

LSS 使用 OD 标识寄存器 （0x1018） 作为唯一值来选择节点。因此，LSS 地址始终由四个 32 位值组成。这也意味着 LSS 依赖于此寄存器实际上是唯一的。（必须在每个canopen节点设备上配置 _vendorID_、_productCode_、_revisionNumber_ 和 _serialNumber_，并且是唯一的）。如图是节点 4 的 OD 标识寄存器描述，其值为 0x00000000 0x00000001 0x00000000 0x00000003。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250808170749.png)

#### 3.4.1 LSS 获取设备节点信息

1. LSS 选中节点4：

	```bash
	cocomm "lss_switch_sel 0x00000000 0x00000001 0x00000000 0x00000003"
	```

	LSS选中节点的报文数据如下，0x7E4 报文返回 0x44，表示 LSS 识别码写入确认的响应：

	![](https://raw.githubusercontent.com/JackHuang021/images/master/LSS选中节点报文数据.png)

2. LSS 获取节点ID：

	```bash
	cocomm "lss_get_node"
	```

	对应的报文数据如下，0x7E4 报文返回的第二个字节 0x04，表示当前设备节点为4：
	![](https://raw.githubusercontent.com/JackHuang021/images/master/LSS获取节点ID.png)

#### 3.4.1 LSS 修改设备节点信息

修改节点 4 的节点ID为 10：

```bash
cocomm "lss_set_node 10"
cocomm "lss_store"
cocomm "lss_switch_glob 0"
cocomm "4 reset comm"
```

下面是报文数据，可以看到节点 4 的节点ID被修改为了 10，在节点 4 复位通讯后，发送的上线报文为 0x70A，表示当前的节点已经修改为10了：

![](https://raw.githubusercontent.com/JackHuang021/images/master/LSS修改节点号.png)

### 3.5 PDO测试

PDO 提供 CANopen 设备对象字典中对象条目的实时数据传输。PDO的传输没有协议开销，即CAN报文中的8个字节长度均为设备的实时数据。PDO 对应于对象字典中的对象，并为应用程序对象提供接口（即PDO报文映射的数据地址）。应用程序对象的数据类型由对象字典中相应的 PDO 映射结构确定。

#### 3.5.1 读取PDO参数

读取节点 4 的 RPDO 配置参数：

```bash
cocomm "set node 4"
cocomm "r 0x1400 1 x32" "r 0x1401 1 x32" "r 0x1402 1 x32" "r 0x1403 1 x32"
cocomm "r 0x1600 0 u8" "r 0x1601 0 u8" "r 0x1602 0 u8" "r 0x1603 0 u8"
```

运行结果如下，读取到4个RPDO的COD-ID分别为0x204、0x304、0x404、0x504，其中RPDO1和RPDO2映射的数据个数分别为2和4，与对象字典中的配置能对应。

![](https://raw.githubusercontent.com/JackHuang021/images/master/读取PDO配置信息.png)

#### 3.5.2 设置PDO参数，查看PDO报文

设置节点 4 的 TPDO1 发送周期为500ms，并查看其PDO报文：

```bash
cocomm "w 0x1800 5 u16 500"
```

对应的报文数据如下，可以看到修改 TPDO1 的发送周期为 500ms 后，0x184 的报文按照 500ms 的周期发送。

![](https://raw.githubusercontent.com/JackHuang021/images/master/设置RPDO1发送周期.png)

