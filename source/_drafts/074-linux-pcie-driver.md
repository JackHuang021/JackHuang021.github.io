---
title: linux pci驱动框架
tags:
---


## 1. PCI总线组成

PCI总线(Peripheral Component Interconnect)，由intel公司提出，其主要功能是连接外部设备

PCI Local Bus的系统架构图

![](https://raw.githubusercontent.com/JackHuang021/images/master/pcie.drawio.png "PCI Local Bus的系统架构图")

1. Host Bridge：图中处理器、Cache、内存子系统通过Host Bridge连接到PC上，Host Bridge管理PCI总线，是联系处理器和PCI设备的桥梁，完成处理器与PCI设备间的数据交换。其中数据交换，包括处理器访问PCI设备的地址空间和PCI设备使用DMA访问主存储器。此外，Host Bridge还可选的支持仲裁机制、热插拔等；
2. PCI Local Bus：PCI总线，由Host Bridge或者PCI-to-PCI Bridge管理，用来连接各类设备，比如声卡、网卡、IDE接口等。可以通过PCI-to-PCI Bridge来扩展PCI总线，并构成多级总线的总线树。比如图中的PCI Local Bus #0和PCI Local Bus #1两条PCI总线就构成一颗总线树，同属一个总线域；
3. PCI-to-PCI Bridge：PCI桥，用于扩展PCI总线，使采用PCI总线进行大规模系统互联成为可能，管理下游总线，并转发上下游总线之间的事务；
4. PCI Device：PCI总线中有三类设备：PCI从设备，PCI主设备，桥设备。PCI从设备被动接收来自Host Bridge或者其他PCI设备的读写请求；PCI主设备可以通过总线仲裁获得PCI总线的使用权，主动向其他PCI设备或主存储器发起读写请求。

## 2. PCIe结构拓扑

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241011095915.png)

1. Root Complex：CPU和PCIE总线之间的接口可能会包含几个模块（处理器接口、DRAM接口等），这个集合就称为Root Complex，它作为PCIe架构的根，代表CPU与系统其它部分进行交互。Root Complex可以认为是CPU和PCIe拓扑之间的接口，Root Complex会将CPU的request转换成PCIe的4种不同的请求（Configuration、Memory、I/O、Message）；
2. Switch：提供扇出能力，让更多的PCIe设备连接在PCIe端口上；
3. Bridge：桥接设备，用于去连接其它的总线，比如PCI总线或PCI-X总线或者另外的PCIe总线；
4. PCIe Endpoint：PCIe设备。

Root Complex通常会实现一个内部总线架构和多个桥，从而扇出到更多接口上；Switch是一个扩展设备，所以看起来像是各种桥的连接路由。

PCIe规范定义了分层的架构设计，包含三层

1. Transcation Layer：负责TLP包（Transaction Layer Packet）的封装与解封装，此外还负责QoS、流控、排序等功能
2. Data Link Layer：负责DLLP包（Data Link Layer Packet）的封装与解封装，此外还负责连接错误检测和校正
3. Physical Layer：负责Ordered-Set包的封装与解封装，物理层处理TLPs、DLLPs、Ordered-Set三种类型的包传输

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241011103251.png)

数据包的封装与解封装，与网络包的创建与解析很类似，如下图：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241011103342.png)

## 3. PCIe设备

PCIe上连接的设备可以分为两种类型：

+ Type 0：表示PCIe上最终端的设备，比如常见的显卡、声卡、网卡
+ Type 1：表示一个PCIe Switch或Root Port，和终端设备不同，它的主要作用是用来连接其它的PCIe设备

### 3.1 PCIe设备地址

PCIe上所有的设备，无论是Type 0还是Type 1，在系统启动的时候，都会被分配一个唯一的地址，它有三个部分组成，一般写作`BB:DD.F`的格式：

+ Bus Number：8 bits，也就是最多256条总线
+ Device Number：5 bits，也就是最多32个设备
+ Function Number：3 bits，也就是最多8个功能

下面是E2000Q demo板上的PCIe设备
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241206100156.png)

由于默认BDF的方式最多只支持8个Function，可能不够用，所以PCIe还支持另一种解析方式，叫做ARI（Alternative Routing-ID Interpretation），它将Device Number和Function Number合并为一个8bit的字段，只用于表示Function，所以最多可以支持256个Function

### 3.2 PCIe Port / PCIe Bridge

### 3.3 PCIe Switch

如果需要连接不止一个设备就需要用到PCIe Switch，PCIe Switch内部主要有三个部分：

+ 1个Upstream Port：用于连接到上游的Port，比如Root Port或者上游Switch的Downstream Port
+ 1组Downstream Port：用于连接下游设备
+ 一根虚拟总线，用于将上游和下游所有的端口连接起来

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241206102018.png)

由于PCIe的信号传输是点对点的，所以Switch中间的这个总线只是一个逻辑上的虚拟的总线，其实并不存在，里面真正的结构是一套用于转发的交换电路

## E2000 PCIe地址空间划分

| 名称          | 地址范围               | 说明                                                                 |
|---------------|------------------------|----------------------------------------------------------------------|
| peu_psu.HPB   | 0x3110_0000~0x3110_0FFF | PEU_PSU HPB自定义寄存器                                              |
| peu.HPB       | 0x3110_1000~0x3110_1FFF | PEU HPB自定义寄存器                                                  |
| peu_psu.ras   | 0x3140_0000~0x3140_0FFF | peu_psu定义的ras寄存器，主要用于记录错误情况                         |
| peu.ras       | 0x3140_1000~0x3140_1FFF | peu定义的RAS寄存器，主要用于记录错误情况                             |
| peu.sriov     | 0x3110_0000~0x3110_7FFF | sriov function定义的寄存器4KB*8                                      |
| peu_psu.c0    | 0x3100_0000~0x3101_FFFF | peu_psu c0控制器寄存器地址                                           |
| peu_psu.c1    | 0x3102_0000~0x3103_FFFF | peu_psu c1控制器寄存器地址                                           |
| peu.c0        | 0x3104_0000~0x3105_FFFF | peu c0控制器寄存器地址                                               |
| peu.c1        | 0x3106_0000~0x3107_FFFF | peu c1控制器寄存器地址                                               |
| peu.c2        | 0x3108_0000~0x3109_FFFF | peu c2控制器寄存器地址                                               |
| peu.c3        | 0x310A_0000~0x310B_FFFF | peu c3控制器寄存器地址                                               |
| peu.phy       | 0x3130_0000~0x313F_FFFF | phy寄存器地址空间                                                    |
| peu.noc       | 0x3150_0000~0x315F_FFFF | peu内部子网络地址空间，主要用于配置安全相关等                        |
| PCIE低空间    | 0x4000_0000~0x7FFF_FFFF | PCIE配置空间，IO空间和mem32空间                                      |
| PCIE高空间    | 0x10_0000_0000~0x1F_FFFF_FFFF | mem64空间                                                            |

## PCIe设备树描述

E2000的pcie设备树描述

```c
pcie: pcie@40000000 {
    compatible = "pci-host-ecam-generic";
    device_type = "pci";
    #address-cells = <3>;
    #size-cells = <2>;
    #interrupt-cells = <1>;
    reg = <0x0 0x40000000 0x0 0x10000000>;
    msi-parent = <&its>;
    bus-range = <0x0 0xff>;
    interrupt-map-mask = <0x0 0x0 0x0 0x7>;
    interrupt-map =
        <0x0 0x0 0x0 0x1 &gic 0x0 0x0 GIC_SPI 4 IRQ_TYPE_LEVEL_HIGH>,
        <0x0 0x0 0x0 0x2 &gic 0x0 0x0 GIC_SPI 5 IRQ_TYPE_LEVEL_HIGH>,
        <0x0 0x0 0x0 0x3 &gic 0x0 0x0 GIC_SPI 6 IRQ_TYPE_LEVEL_HIGH>,
        <0x0 0x0 0x0 0x4 &gic 0x0 0x0 GIC_SPI 7 IRQ_TYPE_LEVEL_HIGH>;
    ranges = <0x01000000 0x00 0x00000000 0x0  0x50000000  0x0  0x00f00000>,
            <0x02000000 0x00 0x58000000 0x0  0x58000000  0x0  0x28000000>,
            <0x03000000 0x10 0x00000000 0x10 0x00000000 0x10  0x00000000>;
    iommu-map = <0x0 &smmu 0x0 0x10000>;
    status = "disabled";
};
```

父节点soc的的`#address_cells = 2`和`#size_cells = 2`，ranges属性值按照`<child-bus-address, parent-bus-address, length>`格式编写：

+ child-bus-address：子总线地址空间的物理地址，由父节点的#address-cells来确定此物理地址所占用的字长
+ parent-bus-address：父总线地址空间的物理地址，由父节点的#address-cells来确定此物理地址所占的字长
+ length：子地址空间的长度，由父节点的#size-cells确定此地址长度所占用的字长


## PCIe BAR空间

PCIe中的每一个设备，无论是Endpoint还是Switch都会分配自己的内存地址空间，这个地址空间会被映射到系统的物理地址空间，并最终映射到虚拟内存中去。当CPU发起一个内存读写请求的时候，如果这个地址经过了MMU的翻译，最后的物理地址落到了PCIe某个设备的内存空间之后，就会触发Root Complex将其转换为PCIe的请求，并通过PCIe总线发给对应的设备。

## 4. pcie配置空间

PCIe的配置空间（Configuration Space）是一种用于存储和管理PCIe设备相关信息的特殊地址空间。它包含了设备的配置寄存器和扩展配置寄存器，这些寄存器用于描述设备的功能、性能、资源分配等信息，设备在出厂时，配置空间是有一些默认值的。

PCIe配置空间的寄存器可以通过将其映射到系统内存地址空间的方式进行访问。通过内存映射，可以使用读写内存的指令来读取和写入配置空间的寄存器。

早期的PCI时期，系统为每个PCI设备分配的内存大小仅有256个Bytes，其中前64字节是标准配置空间header，后面的192字节是Capability结构， 展示pci能提供的能力。到后来的PCIE时期，PCIe设备性能增强，PCIe设备的配置空间扩展至4K Bytes。但为了兼容PCI，PCIe的配置空间前256字节与PCI保持一致，256~4096字节是pcie 扩展配置空间。PCIe一共支持256条Bus,每条bus支持32个Dev，每个dev支持8个Func。因此在满负载的情况下，共需内存大小4k * 256 328 = 256MB

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241206095318.png)

## 3. PCI寻址

`/proc/iomem`描述了系统中所有设备I/O在内存地址空间上的映射，下面是E2000的内存地址空间

```bash
root@Ubuntu:~# cat /proc/iomem
00000000-0ffffffe : qspi_mm
28000000-28000fff : mmc@28000000
28001000-28001fff : mmc@28001000
28003000-28003fff : ddma@28003000
28004000-28004fff : ddma@28004000
28005000-28005fff : i2s@28009000
28008000-28008fff : qspi
28009000-28009fff : i2s@28009000
2800a000-2800afff : can@2800a000
2800b000-2800bfff : can@2800b000
2800d000-2800dfff : uart@2800d000
  2800d000-2800dfff : uart@2800d000
2800e000-2800efff : uart@2800e000
  2800e000-2800efff : uart@2800e000
28026000-28026fff : i2c@28026000
28030000-28030fff : i2c@28030000
28034000-28034fff : gpio@28034000
28035000-28035fff : gpio@28035000
28036000-28036fff : gpio@28036000
28037000-28037fff : gpio@28037000
28038000-28038fff : gpio@28038000
28039000-28039fff : gpio@28039000
2803c000-2803cfff : spi@2803c000
28040000-28040fff : watchdog@28040000
28041000-28041fff : watchdog@28040000
28042000-28042fff : watchdog@28042000
28043000-28043fff : watchdog@28042000
30000000-307fffff : iommu@30000000
31a08000-31a1ffff : usb3@31a08000
31a28000-31a3ffff : usb3@31a28000
31a40000-31a40fff : sata@31a40000
32000000-32007fff : dc@32000000
32008000-32008fff : i2s_dp0@32009000
32009000-32009fff : i2s_dp0@32009000
3200a000-3200afff : i2s_dp1@3200B000
3200b000-3200bfff : i2s_dp1@3200B000
3200c000-3200dfff : ethernet@3200c000
32014000-32014fff : sata@32014000
32a00000-32a00fff : mailbox@32a00000
32a10000-32a11fff : 32a10000.sram
32a36000-32a36fff : rng@32a36000
32b34000-32b34fff : gdma@32b34000
40000000-4fffffff : PCI ECAM
58000000-7fffffff : pcie@40000000
  58000000-581fffff : PCI Bus 0000:01
    58000000-58003fff : 0000:01:00.0
      58000000-58003fff : nvme
  58200000-583fffff : PCI Bus 0000:02
80000000-fbffffff : System RAM
  80000000-8000ffff : reserved
  90280000-9117ffff : Kernel code
  91180000-9122ffff : reserved
  91230000-91343fff : Kernel data
  f5c30000-f9c37fff : reserved
  f9ff7000-f9ffefff : reserved
  f9fff000-fbffffff : reserved
1000000000-1fffffffff : pcie@40000000
  1000000000-10000fffff : 0000:00:01.0
  1000100000-10002fffff : PCI Bus 0000:01
  1000300000-10003fffff : 0000:00:02.0
  1000400000-10005fffff : PCI Bus 0000:02
2000000000-217fffffff : System RAM
  2177000000-217effffff : reserved
  217f30f000-217ffc6fff : reserved
  217ffc9000-217ffccfff : reserved
  217ffcd000-217ffcdfff : reserved
  217ffce000-217ffd0fff : reserved
  217ffd1000-217fffefff : reserved
  217ffff000-217fffffff : reserved
```

`58000000-58003fff : 0000:01:00.0`，这是一个pcie设备的内存地址空间映射，`58000000-58003fff`是它所映射的内存地址空间，占据了内存地址空间的4KB大小，`0000:01:00.0`表示一个pcie外设的地址，它分为4部分，第一部分16位长度表示域，第二部分8位长度表示一个总线编号，第三部分5位长度表示一个设备号，第四部分3位长度表示功能号

因为PCI规范允许单个系统拥有高达256个总线，所以总线编号是8位。但对于大型系统而言，这是不够的，所以，引入了域的概念，每个PCI域可以拥有最多256个总线，每个总线上可支持32个设备，所以设备号是5位，而每个设备上最多可有8种功能，所以功能号是3位。由此，我们可以得出上述的PCI设备的地址是0号域1号总线上的0号设备上的0号功能。

在E2000上使用lspci查看这个设备对应的是哪一个pcie设备

```bash
root@Ubuntu:~# lspci
00:01.0 PCI bridge: Phytium Technology Co., Ltd. Device dc01
00:02.0 PCI bridge: Phytium Technology Co., Ltd. Device dc01
01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller 980
```

lspci没有标明域，对于一台PC一般只有一个域，即0号域。通过上面lspci的输出可以看到这个设备是一个PCI桥。并且可以看到E2000Q开发板上总共有2个PCI总线，在单个系统上，多个总线是通过PCI桥来连接的，PCI系统的整体布局为树形，可以通过`lspci -vt`打印出PCI的树形结构

```bash
root@Ubuntu:~# lspci -vt
-[0000:00]-+-01.0-[01]----00.0  Samsung Electronics Co Ltd NVMe SSD Controller 980
           \-02.0-[02]--
```

0000:00 表示根总线（Root Bus），也就是 PCI 根控制器所在的总线。0000 是域号（Domain Number），表示此总线所属的域。00 是总线号（Bus Number），表示该总线的编号。

`+-01.0-[01]----00.0`，01.0：这是连接到根总线 0000:00 的设备的编号。01 是设备号（Device Number），0 是功能号（Function Number），也就是说，这是设备 01 的功能 0。-[01]：括号中的 01 表示这是一条二级总线（Secondary Bus），连接到设备0000:00:01.0，其总线号为 01。----00.0：在此总线（01）上，有一个 PCI 设备，其设备号为 00，功能号为 0。

![](https://raw.githubusercontent.com/JackHuang021/images/master/e2000_pcie.drawio.png)

## 3. lspci使用

参数说明

| 参数 | 说明 |
| :-: | :- |
| -v | 显示所有pcie设备的一些信息 |
| -vv | 显示更多的信息，几乎包括了所有有用的信息 |
| -vvv | 显示相当详细的信息，所有能够解析出来的pcie信息都会显示出来 |
| -n | 显示所有pcie设备的vendor id和device id |
| -x | 显示所有pcie设备配置空间的头部分 前64字节 |
| -xxx | 显示所有pcie设备配置空间的所有内容 |
| -xxxx | 显示PCI-X 2.0和PCIe总线上扩展配置空间的内容 |
| -b | 显示pcie设备的总线地址 |
| -t | 以树形结构显示pcie设备，展示所有pcie总线、桥、pcie设备之间的连接关系 |
| -s domain:bus:slot.func | 根据domain bus号信息，查看指定pcie设备的信息 |
| -d device_id:vendor_id | 查看指定device id和vendor id的pcie设备的信息 |

## 4. 链路训练状态机 LTSSM

LTSSM全称为Link Training and Status State Machine，是PCIe物理层实现的，用于控制和管理PCIe总线上的数据链路。它提供了一组状态，以便设备进行链路训练和链接协商。在PCIe总线上，发送端和接收端需要进行链路训练，以便确定最佳的链接速度和链接宽度。LTSSM的作用是控制这个过程，并在链路训练期间跟踪链路状态和错误。

LTSSM状态包括：Detect、Polling、Configuration、Recovery、L0、L0s、L1、L2、Hot Reset、Loopback和Disable。当设备之间开始建立连接时，LTSSM从Detect状态开始。然后，它进入Polling状态，等待对方回应确认连接。如果确认完成，则进入Configuration状态，进行链路配置。之后，LTSSM进入L0状态，表明链路处于活动状态。如果设备需要低功耗状态，则可以进入L0s或L1状态。如果出现错误，则可能会进入L2状态或Loopback状态进行修复。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241028173614.png)

LTSSM的11个状态，可以分为下面4个类型

1. PCIe链路训练状态： Detect、Polling 和 Configuration状态。正常PCIe链路训练状态转换流程：Detect -> Polling -> Configuration -> L0；L0是PCIe链路可以正常工作的电源状态。
2. PCIe链路重训练状态： Recovery 状态。进入这个状态因素很多，比如电源状态的变化，PCIe链路速率的变化等
3. 电源管理状态： 基于硬件控制的ASPM（Active State Power Management）电源管理机制，是基于硬件自主控制的链路电源管理机制，只有在PCIe设备处于D0状态时才可以启动ASPM机制，与ASPM有关的链路状态有L0、L0s、L1 （包括L1.1和L1.2）和 L2
4. 其它状态： Disable、Loopback 和 Hot Reset

### 4.1 链路训练相关知识

1. 位锁定（bit lock）： 因为PCIe总线在进行数据传输时需要使用时钟进行同步，但是PCIe链路中并未提供这个时钟信号，因此进行链路训练时接收端需要从发送端的数据报文中提取接收时钟，这个过程被称为位锁定。
2. 字符锁定（symbol lock）： 在链路训练过程中，PCIe链路要首先确定COM字符，它标志着链路训练开始或者重新训练的开始，确定COM字符的标志被称为字符锁定。

### 4.1 Detect状态

Detect状态主要的作用是用来确认PCIe链路上可以正常工作的lane资源。

Detect状态是在基本复位或者软件产生的热复位命令后进入的初始状态，在复位80ms内进入这个状态。Detect状态也能从其它状态进入。Detect有两个子状态：Detect.Quiet和Detect.Active。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241030102616.png)

进入Detect状态后首先进入Dectect.Quiet子状态，当处于Detect.Quiet子状态时间超过12ms或者链路退出Electrical Idle状态时，进入到Detect.Active状态。进入Detect.Active状态后发送端发送Receiver Detection Sequence来检查是否有接收端，检查到了接收端则进入Polling状态，否则返回Detect.Quiet状态，等待12ms后进行循环检测。

### 4.2 Polling状态

Polling状态主要功能是获取bit/symbol lock以及同步链路两端使其进入下一个config状态。

Polling状态一共有3个子状态：

+ Polling.Active：判断对端的 Lane 数量和链路速率等基本信息，并准备进入下一个子状态。
+ Polling.Compliance：通常用于测试设备的信号质量、电气特性以及协议兼容性
+ Polling.Configuration：链路两端通过交换TS1和TS2序列协商具体的链路参数

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241030104239.png)

#### 4.2.1 TS1 TS2训练序列

TS1/2 由16个字节组成。在LTSSM的轮询、配置以及恢复状态中，通信双方会交换TS1/2序列。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241030145040.png)

#### 4.2.2 link number和lane number的区别

PCI总线是一种地址和数据复用的总线，地址和数据占用同一组信号线。PCIe采用了差分、全双工的传输设计，设备之间通过双向的link连接，每个link支持1-32个通道（lane）

+ link number： 指的是两个PCIe部件的连接，通常是由端口和lane组成。
+ lane number： 指的是一组差分信号的组合，包括发送和接收，一个发送方向的差分信号包括TX+和TX-两条线，所以一组lane有4条物理连线。

#### 4.2.1 Polling.Avtive

Polling状态中进行链路训练，确定链路数量和链路速率。发送端在Detect状态监测到的接收端的所有lane上发送TS1序列（TS1序列和TS2序列 参考《PCI Express Base 4.0 Rev0.3》的4.2.4.1 Training Sequence章节）

以下任意一个条件成立，则从Polling.Active进入到Polling.Cofiguration状态，否则进入polling.compliance状态对链路进行修复：

1. 1024个TS1序列全部发送，且在Detect状态检测到的接收端的所有lane上都收到8个连续的训练序列满足lane和link number字段均被填充，且Compliance Receive bit (bit 4 of Symbol 5) 为0或Loopback bit (bit 2 of Symbol 5)为1
2. 达到24ms超时时间，且在Detect状态检测到的接收端的任一lane上都收到8个连续的训练序列满足lane和link number字段均被填充，且Compliance Receive bit (bit 4 of Symbol 5) 为0或Loopback bit (bit 2 of Symbol 5)为1，并且至少Detect状态检测到接收端的所有lane检测到一次退出Electrical Idle的情况

#### 4.2.2 Polling.Configuration

接收端按需进行极性翻转，发送端在所有lane上发送TS2序列，并且link和lane number均被填充为PAD字段。接收端连续收到8个TS2序列，且link和lane number字段均被填充则转入到Configuration状态；否则超过48ms后转入到Detect状态

### 4.3 Configuration

Configuration状态主要用于链路参数的进一步确认和配置（如lane数量，链路速率等），Configuration 状态通过多次交换 TS1 和 TS2 训练序列来进行链路参数的配置和确认，最终为进入 L0（Link Up）状态做好准备。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241030140559.png)

在Configuration状态中，Downstream和Upstream是描述数据流向的两个重要概念。

+ Downstream： 指的是数据从RC传输到EP的方向，可以理解为下游数据传输，Downstream通常涉及RC向EP发送配置数据、控制信息和训练序列。
+ Upstream： 指的是数据从EP传输回RC的方向，可以理解为上游数据传输

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031142129.png)

#### 4.3.1 Configuration.LinkWidth.x

通过互相发送TS1来确定PCIe下游端口的link num拓扑结构

+ Downstream： 发送端在所有lane上发送TS1序列，序列中link number需要被设定，lane number使用PAD进行填充。接收端收到两个连续的TS1序列，且link number不为PAD，且和之前发送的link number能够匹配则进入Configuration.LinkWidth.Accept子状态。所有lane接着发送TS1序列，且link number和lane number均被正确设置。

+ Upstream： 发送端在所有lane上发送TS1序列，序列中link number和lane number设置为PAD填充。当接收端收到两个连续的TS1序列，且link number不为PAD，lane number为PAD，则进入Configuration.LinkWidth.Accept子状态。收到连续两个TS1序列，且link number和lane number均不为PAD，Upstream再继续发送TS1序列，根据Upstream自身情况设置lane number。

#### 4.3.2 Configuration.Lanenum.x

## 5. ASPM

### 5.1 ASPM介绍

ASPM（Active State Power Management）在pcie设备在D0状态下通过将链路进入低功耗状态，这个过程由硬件自主完成。

ASPM对应两个低功耗的链路状态：

+ L0s：在提供显著的节能效果下，并且可以低时延的进入和退出该状态，支持L0s的功能是可选的
+ L1：可以提供最大程度的节能效果，但是进入和退出存在高时延，另外还包括两个子状态L1.1和L1.2，在需要非常低的空闲功率并且可以接受更长的转换时间的情况下，这些子状态可以进一步降低链路功率

PCIe设备需要在配置寄存器中描述ASPM的支持情况，以及L0s和L1的退出时延，驱动可能会参考这些时延来决定进入哪种ASPM链路状态（时延过高可能导致数据处理上的问题），ASPM功能可以按照需求配置关闭和开启

在多功能的PCIe设备中，若其中有一个功能禁用了ASPM，且该功能处于D0状态，则必须要禁用ASPM功能，当该功能处于非D0状态时，则可以开启ASPM功能。

### 5.3 ASPM配置

1. PCIe设备的每个功能都需要配置ASPM的支持情况
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031142657.png)

2. 配置端口时钟源的使用情况，Slot Clock和Common Clock

3. 配置L0s和L1的退出延时
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031143706.png)
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031143727.png)

4. 配置ASPM的使能控制
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031144531.png)

### 5.4 L1子状态

+ L1.1： 维持共模电压（共模电压是指在一个差分信号对中，两个信号线相对于地的电压平均值。），端口不再检查Electrical Idle状态
+ L1.2： 不需要维持共模电压

这两个状态相当于L1状态的进阶，具有更高的功耗降低潜力，但也伴随着更长的恢复时间

### 5.5 查看ASPM的状态

linux下可以通过lspci可以查看ASPM的使能状态，以及ASPM支持的链路状态。

这里以E2000Q demo板为例，查看ASPM的支持状态，开发板上连接的pcie设备有一个nvme ssd和一张intel pcie千兆网卡：

pcie的拓扑结构如下

```bash
root@Ubuntu:~# lspci -vt
-[0000:00]-+-01.0-[01]----00.0  Samsung Electronics Co Ltd NVMe SSD Controller 980
           \-02.0-[02]----00.0  Intel Corporation 82574L Gigabit Network Connection
```

首先查看pcie控制器的ASPM支持情况：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031150421.png)
可以看到ASPM支持的链路状态为L0s和L1 L1.1 L1.2，L0s的退出时延为小于64ns，L1的退出时延为小于1us，此时ASPM功能是关闭的。

再查看intel pcie千兆网卡的ASPM支持情况：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031104937.png)
ASPM支持的链路状态为L0s和L1，ASPM功能是关闭的。

nvme ssd的ASPM支持情况：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031105245.png)
可以看到ASPM支持的链路状态为L0s和L1 L1.1 L1.2，ASPM功能是关闭的。

### 5.6 linux pcie aspm配置

linux需要打开CONFIG_PCIEASPM以支持对aspm的配置，在phytium_defconfig中默认是开启的。

#### 5.6.1 aspm的策略配置

查看内核支持的aspm策略：

```bash
root@Ubuntu:/sys/module/pcie_aspm/parameters# cat policy
[default] performance powersave powersupersave
```

+ defualt: 使用固件的aspm设置
+ powersave: 内核会打开L0s，以及L1
+ performance: 强行关闭L0s,L1,就算BIOS打开了
+ powersuperave: 比powersave多了L1 substate

phytium_defconfig中选中的是default，即使用固件的aspm设置，E2000的固件配置aspm是关闭的。

#### 5.6.2 强制开启/关闭aspm

在启动参数增加"pcie_aspm=force"来强制开启aspm，或增加"pcie_aspm=off"来关闭aspm

### 5.7 查看pcie控制器的链路状态

以phytium E2000Q处理器为例，查看pcie控制器的链路状态。E2000Q 集成的 PCIe 控制器包含PSU模块的2个PCIe控制器与PEU模块的4个PCIe控制器。链路状态寄存器(REG_Cx_LTSSM)在HPB寄存器组中，HPB寄存器组的基地址和链路状态寄存器的偏移如下图：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241028151634.png)

链路状态寄存器的说明如下图：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241028151923.png)

在E2000Q demo板上psu模块的C1控制器的链路状态寄存器的地址为：0x31100544，该pcie控制器上连接了nvme ssd，下面是没有开启ASPM功能的情况下该pcie控制器的链路状态，即为L0状态：

```bash
root@Ubuntu:~# devmem2 0x31100544
/dev/mem opened.
Memory mapped at address 0xffffb0ae9000.
Value at address 0x31100544 (0xffffb0ae9544): 0x10
```

开启aspm后，再次查看其链路状态，进入了L1状态：

```bash
root@Ubuntu:~# echo powersave > /sys/module/pcie_aspm/parameters/policy 
root@Ubuntu:~# devmem2 0x31100544
/dev/mem opened.
Memory mapped at address 0xffffa7465000.
Value at address 0x31100544 (0xffffa7465544): 0x13
```

### 5.8 使用setpci控制aspm

通过设置Link Control Register来控制aspm功能的关闭/开启，该寄存器的偏移是0x10
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031153954.png)

以E2000Q demo板上接的intel pcie千兆网卡为例，通过setpci来开启/关闭aspm功能。

1. 查看当前pcie网卡对应RC端的Link Control寄存器的值，其地址偏移为0x80+0x10，可以看到值为0x00，即aspm功能是关闭的
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241031160120.png)

2. 设置仅开启L0s，我们设置0x90为0x01，可以看到ASPM L0s开启了

```bash
root@Ubuntu:~# lspci -xxx -s 00:02.0
00:02.0 PCI bridge: Phytium Technology Co., Ltd. Device dc01
00: b7 1d 01 dc 07 00 10 00 00 00 04 06 08 00 01 00
10: 0c 00 30 00 10 00 00 00 00 02 02 00 21 21 00 00
20: 20 58 30 58 41 00 51 00 10 00 00 00 10 00 00 00
30: 00 00 00 00 80 00 00 00 00 00 00 00 49 01 02 00
40: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
50: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
80: 10 d0 42 01 21 80 00 00 1f 29 00 00 43 0c 70 01
90: 01 00 11 60 62 00 00 00 38 14 00 00 08 00 00 00
a0: 00 00 00 00 1f 08 10 00 00 04 00 00 0e 00 00 00
b0: 43 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
d0: 11 e0 00 00 00 00 00 00 08 00 00 00 00 00 00 00
e0: 05 f8 8a 01 00 00 00 00 00 00 00 00 00 00 00 00
f0: 00 00 00 00 00 00 00 00 01 00 03 fe 08 00 00 00

root@Ubuntu:~# lspci -vvv -s 00:02.0 | grep -i aspm
		LnkCap:	Port #1, Speed 8GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <64ns, L1 <1us
			ClockPM- Surprise- LLActRep+ BwNot+ ASPMOptComp+
		LnkCtl:	ASPM L0s Enabled; RCB 64 bytes, Disabled- CommClk-
		L1SubCap: PCI-PM_L1.2+ PCI-PM_L1.1- ASPM_L1.2+ ASPM_L1.1+ L1_PM_Substates+
		L1SubCtl1: PCI-PM_L1.2- PCI-PM_L1.1- ASPM_L1.2- ASPM_L1.1-
root@Ubuntu:~# 
```

### PME 中断注册流程



## 5. PCIe控制器EP驱动

在 Linux 上为一个 PCIe 控制器编写终端设备（Endpoint, EP）驱动，主要涉及到两大部分：

1. PCIe 终端控制器（Endpoint Controller, EPC）驱动：这是控制和管理硬件 PCIe 控制器的驱动，用于配置 PCIe 的 BAR、MSI/MSI-X 等，管理主机与终端设备的交互。
2. PCIe 终端功能（Endpoint Function, EPF）驱动：这是处理 PCIe 设备功能的驱动，包括设备的具体逻辑、内存映射、DMA 传输等。

### 5.1 PCIe控制器EPC驱动的开发

EPC 驱动的作用是控制底层硬件，使其能够作为 PCIe 终端设备与主机通信。Linux 提供了 PCIe 终端控制器框架（pci_epc），开发者需要基于该框架实现硬件相关操作。

实现EPC操作集`pci_epc_ops`，`pci_epc_ops` 是一个函数指针结构体，它定义了控制器所需的各种操作，需要实现这些函数来管理 EPC 的硬件操作。

```c
// include/linux/pci-epc.h
/**
 * struct pci_epc_ops - set of function pointers for performing EPC operations
 * @write_header: ops to populate configuration space header
 * @set_bar: ops to configure the BAR
 * @clear_bar: ops to reset the BAR
 * @map_addr: ops to map CPU address to PCI address
 * @unmap_addr: ops to unmap CPU address and PCI address
 * @set_msi: ops to set the requested number of MSI interrupts in the MSI
 *	     capability register
 * @get_msi: ops to get the number of MSI interrupts allocated by the RC from
 *	     the MSI capability register
 * @set_msix: ops to set the requested number of MSI-X interrupts in the
 *	     MSI-X capability register
 * @get_msix: ops to get the number of MSI-X interrupts allocated by the RC
 *	     from the MSI-X capability register
 * @raise_irq: ops to raise a legacy, MSI or MSI-X interrupt
 * @start: ops to start the PCI link
 * @stop: ops to stop the PCI link
 * @owner: the module owner containing the ops
 */
struct pci_epc_ops {
    // 写入 PCIe 配置空间的头部信息。通过这个函数，PCIe 控制器可以向RC暴露它的设备信息（如设备 ID、厂商 ID）。
	int	(*write_header)(struct pci_epc *epc, u8 func_no,
				struct pci_epf_header *hdr);
    // 设置终端设备的 BAR（Base Address Register），它定义了主机如何访问终端设备的内存。
	int	(*set_bar)(struct pci_epc *epc, u8 func_no,
			   struct pci_epf_bar *epf_bar);
	void	(*clear_bar)(struct pci_epc *epc, u8 func_no,
			     struct pci_epf_bar *epf_bar);
    // 将物理内存映射到 PCIe 地址空间
	int	(*map_addr)(struct pci_epc *epc, u8 func_no,
			    phys_addr_t addr, u64 pci_addr, size_t size);
	void	(*unmap_addr)(struct pci_epc *epc, u8 func_no,
			      phys_addr_t addr);
	int	(*set_msi)(struct pci_epc *epc, u8 func_no, u8 interrupts);
	int	(*get_msi)(struct pci_epc *epc, u8 func_no);
	int	(*set_msix)(struct pci_epc *epc, u8 func_no, u16 interrupts);
	int	(*get_msix)(struct pci_epc *epc, u8 func_no);
    // 触发中断（如 MSI 或者传统的 PCIe 中断）
	int	(*raise_irq)(struct pci_epc *epc, u8 func_no,
			     enum pci_epc_irq_type type, u16 interrupt_num);
	int	(*start)(struct pci_epc *epc);
	void	(*stop)(struct pci_epc *epc);
	struct module *owner;
};
```

PCIe EPC 驱动的工作流程：

1. 硬件初始化：当系统启动时，EPC 驱动被加载并初始化 EPC 硬件。这通常包括配置 PCIe 链路、设置设备的配置空间和初始化所需的硬件资源。
2. 与 Root Complex 建立连接：EPC 驱动通过 PCIe 链路与 Root Complex 进行握手，建立通信。在此过程中，EPC 驱动会处理链路建立和设备的配置请求。
3. 设备配置：Root Complex 会向 EPC 发送配置请求，EPC 驱动负责解析这些请求并对设备进行配置，例如分配内存地址、启用设备等。
4. 处理 I/O 事务：当 Root Complex 发起 I/O 请求（如读写数据）时，EPC 驱动会根据请求内容执行相应的操作，例如读取或写入内存。
5. 中断处理：EPC 驱动还会处理来自 Root Complex 的中断信号，通常用于通知终端设备进行某些操作。

注册EPC驱动

```c
struct pci_epc *epc;

epc = devm_pci_epc_create(&pdev->dev, &my_pci_epc_ops);
```

### 5.2 E2000 PCIe EP设备树节点

```c
pcie_ep: ep@31040000 {
    compatible = "phytium,pd2008-pcie-ep";
    //使用的控制器为 PEU.C0
    reg = <0x0 0x31040000 0x0 0x10000>,
    // 内存空间地址，大小4GB
    <0x11 0x00000000 0x1 0x00000000>,
    // PEU内HPB寄存器地址
    <0x0 0x31101000 0x0 0x1000>;
    reg-names = "reg", "mem", "hpb";
    max-outbound-regions = <3>;
    max-functions = /bits/ 8 <1>;
    status = "disabled";
};
```

### 5.3 PCIe EP功能测试

EP端启动参数

```bash
setenv bootargs 'console=ttyAMA1,115200 audit=0 earlycon=pl011,0x2800d000 root=/dev/sda3 rw'; fatload scsi 0:1 0x90100000 4_19/Image; fatload scsi 0:1 0x90000000 4_19/e2000q-demo-board.dtb;

setenv bootargs 'console=ttyAMA1,115200 audit=0 earlycon=pl011,0x2800d000 root=/dev/sda3 rw'; fatload scsi 0:1 0x90100000 5_10/Image; fatload scsi 0:1 0x90000000 5_10/e2000q-demo-board.dtb;

fdt addr 0x90000000;
fdt rm /iommu;
fdt rm /soc/pcie;

booti 0x90100000 - 0x90000000

modprobe pci-epf-test
mkdir /sys/kernel/config/pci_ep/functions/pci_epf_test/func1
echo 0x16c3 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/vendorid
echo 0xedda > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/deviceid
echo 32 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/msi_interrupts

ln -s /sys/kernel/config/pci_ep/functions/pci_epf_test/func1 /sys/kernel/config/pci_ep/controllers/31040000.ep/

echo 1 > /sys/kernel/config/pci_ep/controllers/31040000.ep/start

ls -l /sys/bus/pci-epf/devices/pci_epf_test.0/


echo 1 > /sys/class/pci_bus/0000\:02/device/remove
echo 1 > /sys/bus/pci/rescan

echo 2 > /sys/bus/pci/devices/0000\:02\:00.0/sriov_numvfs
```
