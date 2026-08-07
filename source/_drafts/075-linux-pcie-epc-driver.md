---
title: 075-linux-pcie-epc-driver
tags:
---

## 1. PCIe控制器EP驱动

E2000 PCIe地址空间划分

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

在 Linux 上为一个 PCIe 控制器编写终端设备（Endpoint, EP）驱动，主要涉及到两大部分：

1. PCIe 终端控制器（Endpoint Controller, EPC）驱动：这是控制和管理硬件 PCIe 控制器的驱动，用于配置 PCIe 的 BAR、MSI/MSI-X 等，管理主机与终端设备的交互。
2. PCIe 终端功能（Endpoint Function, EPF）驱动：这是处理 PCIe 设备功能的驱动，包括设备的具体逻辑、内存映射、DMA 传输等。

## 2 PCIe控制器EPC驱动的开发

EPC 驱动的作用是控制底层硬件，使其能够作为 PCIe 终端设备与主机通信。Linux 提供了 PCIe 终端控制器框架（pci_epc），开发者需要基于该框架实现硬件相关操作。

实现EPC操作集`pci_epc_ops`，它定义了控制器所需的各种操作，需要实现这些函数来管理 EPC 的硬件操作。

```c
// linux6.6 include/linux/pci-epc.h
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
 * @map_msi_irq: ops to map physical address to MSI address and return MSI data
 * @start: ops to start the PCI link
 * @stop: ops to stop the PCI link
 * @get_features: ops to get the features supported by the EPC
 * @owner: the module owner containing the ops
 */
struct pci_epc_ops {
	// 写入 PCIe 配置空间的头部信息。通过这个函数，PCIe 控制器可以向RC暴露它的设备信息（如设备 ID、厂商 ID）。
	int	(*write_header)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
				struct pci_epf_header *hdr);
	// 设置终端设备的 BAR（Base Address Register），它定义了主机如何访问终端设备的内存。
	int	(*set_bar)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			   struct pci_epf_bar *epf_bar);
	void	(*clear_bar)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			     struct pci_epf_bar *epf_bar);
	// 将物理内存映射到 PCIe 地址空间
	int	(*map_addr)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			    phys_addr_t addr, u64 pci_addr, size_t size);
	void	(*unmap_addr)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			      phys_addr_t addr);
	int	(*set_msi)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			   u8 interrupts);
	int	(*get_msi)(struct pci_epc *epc, u8 func_no, u8 vfunc_no);
	int	(*set_msix)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			    u16 interrupts, enum pci_barno, u32 offset);
	int	(*get_msix)(struct pci_epc *epc, u8 func_no, u8 vfunc_no);
	// 触发中断（如 MSI 或者传统的 PCIe 中断）
	int	(*raise_irq)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			     enum pci_epc_irq_type type, u16 interrupt_num);
	int	(*map_msi_irq)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
			       phys_addr_t phys_addr, u8 interrupt_num,
			       u32 entry_size, u32 *msi_data,
			       u32 *msi_addr_offset);
	int	(*start)(struct pci_epc *epc);
	void	(*stop)(struct pci_epc *epc);
	const struct pci_epc_features* (*get_features)(struct pci_epc *epc,
						       u8 func_no, u8 vfunc_no);
	struct module *owner;
};
```

PCIe EPC 驱动的工作流程：

1. 硬件初始化：当系统启动时，EPC 驱动被加载并初始化 EPC 硬件，包括配置 PCIe 链路、设置设备的配置空间和初始化所需的硬件资源。
2. 与 Root Complex 建立连接：EPC 驱动通过 PCIe 链路与 Root Complex 进行握手，建立通信。
3. 设备配置：Root Complex 会向 EPC 发送配置请求，EPC 驱动负责解析这些请求并对设备进行配置，例如分配内存地址、启用设备等。
4. 处理 I/O 事务：当 Root Complex 发起 I/O 请求（如读写数据）时，EPC 驱动会根据请求内容执行相应的操作，例如读取或写入内存。
5. 中断处理：EPC 驱动还会处理来自 Root Complex 的中断信号，通常用于通知终端设备进行某些操作。

注册EPC驱动

```c
static const struct pci_epc_ops phytium_pcie_epc_ops = {
	.write_header	= phytium_pcie_ep_write_header,
	.set_bar	= phytium_pcie_ep_set_bar,
	.clear_bar	= phytium_pcie_ep_clear_bar,
	.map_addr	= phytium_pcie_ep_map_addr,
	.unmap_addr	= phytium_pcie_ep_unmap_addr,
	.set_msi	= phytium_pcie_ep_set_msi,
	.get_msi	= phytium_pcie_ep_get_msi,
	.raise_irq	= phytium_pcie_ep_raise_irq,
	.start		= phytium_pcie_ep_start,
};

struct pci_epc *epc;

epc = devm_pci_epc_create(&pdev->dev, &phytium_pcie_epc_ops);
```

E2000 PCIe EP设备树节点

```c
pcie_ep: ep@31040000 {
	compatible = "phytium,pcie-ep-2.0";
	//使用的控制器为 PEU.C0
	reg = <0x0 0x31040000 0x0 0x10000>,
	// 内存空间地址，大小4GB
	<0x11 0x00000000 0x1 0x00000000>,
	// PEU内HPB寄存器地址
	<0x0 0x31101000 0x0 0x1000>;
	reg-names = "reg", "mem", "hpb";
	// 定义了EPC设备可以配置的最大外部内存区域数量
	max-outbound-regions = <3>;
	// PF个数配置为1
	max-functions = /bits/ 8 <1>;
	// PF对应VF的个数配置
	max-virtual-functions = /bits/ 8 <1>;
	status = "okay";
};
```

### 飞腾EPC驱动

`struct phytium_pcie_ep`结构体

```c
// drivers/pci/controller/pcie-phytium-ep.h
struct phytium_pcie_ep {
	/* pcie 控制器寄存器基地址 */
	void __iomem		*reg_base;
	struct resource		*mem_res;
	void __iomem		*hpb_base;
	u32 			hpb_perf_base_limit_offs;
	unsigned int		max_regions;
	unsigned long		ob_region_map;
	phys_addr_t		*ob_addr;
	phys_addr_t		irq_phys_addr;
	void __iomem		*irq_cpu_addr;
	unsigned long		irq_pci_addr;
	u8			irq_pci_fn;
	struct pci_epc		*epc;
};
```

## 3. PCIe EP的SR-IOV功能支持

SR-IOV（Single Root I/O Virtualization）是一种PCI Express（PCIe）技术，旨在提高虚拟化环境中I/O设备的效率和灵活性。SR-IOV允许单个PCIe设备在物理层面上划分为多个虚拟功能（Virtual Functions，VF）。每个VF可以被独立分配给虚拟机（VM），从而实现直接访问硬件。

SR-IOV的实现有以下几个部分组成：
1. Single Root PCI Manager(SR-PCIM)：负责配置 SRIOV 功能、管理物理功能和虚拟功能的软件
2. Optional Translation Agent(TA)：负责将 PCIe 事务中的地址转换为相关平台物理地址
3. Optional Address Translation and Protection Table (ATPT) :
4. Optional Address Translation Cache (ATC)：
5. Optional Access Control Services (ACS)：
6. Physical Function (PF)：
7. Virtual Function (VF)：

### SR-IOV配置流程

1. 配置SR-IOV Capability：这一步在RC端进行
2. 配置VF的BAR

E2000的编程手册上描述支持SR-IOV功能，从lspci打印出的信息来看，硬件是支持SR-IOV的，最大支持的VF为8个
![](https://raw.githubusercontent.com/JackHuang021/images/master/sriov特性.png)

在RC端可以正常创建VF设备
![](https://raw.githubusercontent.com/JackHuang021/images/master/vf功能测试.png)

需要参考linux主线补丁，在phytium epc驱动中添加对VF的配置，
[Add SR-IOV support in PCIe Endpoint Core](https://patchwork.kernel.org/project/linux-rockchip/cover/20210419083401.31628-1-kishon@ti.com/)

目前我们的内核只有linux 6.6合入了这笔补丁，要将phytium epc驱动移植到linux 6.6内核，还需要对`phytium_pcie_epc_ops`的接口做一些修改，主要是来源这笔patch的改动: [PCI: endpoint: Add virtual function number in pci_epc ops](https://patchwork.kernel.org/project/linux-rockchip/patch/20210419083401.31628-5-kishon@ti.com/)。

### 3.1 SR-IOV功能测试

修改设备树EP节点，增加max-virtual-functions属性，E2000的设备树节点如下

```c
pcie_ep: ep@31040000 {
	compatible = "phytium,pcie-ep-2.0";
	//使用的控制器为 PEU.C0
	reg = <0x0 0x31040000 0x0 0x10000>,
	// 内存空间地址，大小4GB
	<0x11 0x00000000 0x1 0x00000000>,
	// PEU内HPB寄存器地址
	<0x0 0x31101000 0x0 0x1000>;
	reg-names = "reg", "mem", "hpb";
	// 定义了EPC设备可以配置的最大外部内存区域数量
	max-outbound-regions = <3>;
	// PF个数配置为1
	max-functions = /bits/ 8 <1>;
	// PF对应VF的个数配置
	max-virtual-functions = /bits/ 8 <1>;
};
```

FT2000-4 设备树节点
P
```c
pcie_ep: ep@29030000 {
	compatible = "phytium,pcie-ep-1.0";
	//使用的控制器为 PEU.C0
	reg = <0x0 0x29030000 0x0 0x10000>,
	// 内存空间地址，大小4GB
	<0x11 0x00000000 0x1 0x00000000>,
	// PEU内HPB寄存器地址
	<0x0 0x29100000 0x0 0x1000>;
	reg-names = "reg", "mem", "hpb";
	// 定义了EPC设备可以配置的最大外部内存区域数量
	max-outbound-regions = <3>;
	// PF个数配置为1
	max-functions = /bits/ 8 <1>;
};
```

对EPC驱动的测试可以参考内核文档`Documentation/PCI/endpoint/pci-endpoint-cfs.rst`

#### 3.1.1 EP端配置

```bash
# 加载pci-epf-test驱动
modprobe pci-epf-test

# 创建PF
mkdir /sys/kernel/config/pci_ep/functions/pci_epf_test/func1
echo 0x16c3 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/vendorid
echo 0xedda > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/deviceid
echo 32 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/msi_interrupts

# 创建一个VF
mkdir /sys/kernel/config/pci_ep/functions/pci_epf_test/func2
echo 0x16c3 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func2/vendorid
echo 0xedda > /sys/kernel/config/pci_ep/functions/pci_epf_test/func2/deviceid
echo 32 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func2/msi_interrupts

# 将VF绑定到PF
ln -s /sys/kernel/config/pci_ep/functions/pci_epf_test/func2 /sys/kernel/config/pci_ep/functions/pci_epf_test/func1

# 将PF绑定到epc device
ln -s /sys/kernel/config/pci_ep/functions/pci_epf_test/func1 /sys/kernel/config/pci_ep/controllers/31040000.ep/
echo 1 > /sys/kernel/config/pci_ep/controllers/31040000.ep/start
```

#### 3.1.2 RC端配置

创建一个vf设备，创建完成后可以看到多出了一个PCIe设备，其vendorId和deviceId和PF设备能对应，同时该设备也自动加载了pci_endpoint_test驱动

```bash
root@Ubuntu: echo 2 > /sys/bus/pci/devices/0000:04:00.0/sriov_numvfs
root@Ubuntu: lspci -n
00:01.0 0604: 1db7:dc01
00:04.0 0604: 1db7:dc01
00:05.0 0604: 1db7:dc01
01:00.0 0108: 144d:a809
02:00.0 ff00: 16c3:edda
03:00.0 ff00: 16c3:edda

root@Ubuntu:~# ls -l /sys/bus/pci/devices/0000\:02\:00.0/driver
lrwxrwxrwx 1 root root 0 Nov 22  2023 /sys/bus/pci/devices/0000:02:00.0/driver -> ../../../../../../../bus/pci/drivers/pci-endpoint-test
root@Ubuntu:~# ls -l /sys/bus/pci/devices/0000\:03\:00.0/driver
lrwxrwxrwx 1 root root 0 Feb 14 16:37 /sys/bus/pci/devices/0000:03:00.0/driver -> ../../../../../../../bus/pci/drivers/pci-endpoint-test
```

+ 使用pcitest对VF设备进行测试

```bash
echo 1 > /sys/bus/pci/rescan
# 测试bar
pcitest -D /dev/pci-endpoint-test.1 -b 0
# 测试msi中断
pcitest -D /dev/pci-endpoint-test.1 -i 1
pcitest -D /dev/pci-endpoint-test.1 -m 1
```

```bash
04:00.0 Unassigned class [ff00]: Synopsys, Inc. EPMockUp
	Physical Slot: 0-3
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 32 bytes
	Interrupt: pin ? routed to IRQ 112
	IOMMU group: 4
	Region 0: Memory at 58730400 (32-bit, non-prefetchable) [size=128]
	Region 2: Memory at 58730000 (32-bit, non-prefetchable) [size=1K]
	Region 4: Memory at 58600000 (32-bit, non-prefetchable) [size=128K]
	Capabilities: [80] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 256 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
			ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset- SlotPowerLimit 0.000W
		DevCtl:	CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
			RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop+
			MaxPayload 128 bytes, MaxReadReq 512 bytes
		DevSta:	CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
		LnkCap:	Port #1, Speed 8GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <64ns, L1 <1us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
		LnkCtl:	ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed 8GT/s (ok), Width x1 (downgraded)
			TrErr- Train- SlotClk- DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+ NROPrPrP- LTR+
			 10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt+ EETLPPrefix-
			 EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
			 FRS- TPHComp- ExtTPHComp-
			 AtomicOpsCap: 32bit- 64bit- 128bitCAS-
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR+ OBFF Disabled,
			 AtomicOpsCtl: ReqEn-
		LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink- Retimer- 2Retimers- DRS-
		LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-
			 Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
			 Compliance De-emphasis: -6dB
		LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete+ EqualizationPhase1+
			 EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
			 Retimer- 2Retimers- CrosslinkRes: unsupported
	Capabilities: [e0] MSI: Enable+ Count=32/32 Maskable- 64bit+
		Address: 00000000fffff040  Data: 0000
	Capabilities: [f8] Power Management version 3
		Flags: PMEClk- DSI- D1+ D2+ AuxCurrent=0mA PME(D0+,D1+,D2+,D3hot+,D3cold+)
		Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
	Capabilities: [100 v1] Vendor Specific Information: ID=1556 Rev=1 Len=008 <?>
	Capabilities: [108 v1] Latency Tolerance Reporting
		Max snoop latency: 0ns
		Max no snoop latency: 0ns
	Capabilities: [110 v1] L1 PM Substates
		L1SubCap: PCI-PM_L1.2+ PCI-PM_L1.1- ASPM_L1.2+ ASPM_L1.1+ L1_PM_Substates+
			  PortCommonModeRestoreTime=10us PortTPowerOnTime=10us
		L1SubCtl1: PCI-PM_L1.2- PCI-PM_L1.1- ASPM_L1.2- ASPM_L1.1-
			   T_CommonMode=0us LTR1.2_Threshold=26016ns
		L1SubCtl2: T_PwrOn=10us
	Capabilities: [120 v1] Address Translation Service (ATS)
		ATSCap:	Invalidate Queue Depth: 0a
		ATSCtl:	Enable-, Smallest Translation Unit: 00
	Capabilities: [128 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS-, Next Function: 0
		ARICtl:	MFVC- ACS-, Function Group: 0
	Capabilities: [130 v1] Page Request Interface (PRI)
		PRICtl: Enable- Reset-
		PRISta: RF- UPRGI- Stopped+
		Page Request Capacity: 00045678, Page Request Allocation: 00000000
	Capabilities: [140 v1] Single Root I/O Virtualization (SR-IOV)
		IOVCap:	Migration-, Interrupt Message Number: 000
		IOVCtl:	Enable+ Migration- Interrupt- MSE+ ARIHierarchy-
		IOVSta:	Migration-
		Initial VFs: 8, Total VFs: 8, Number of VFs: 1, Function Dependency Link: 00
		VF offset: 256, stride: 1, Device ID: edda
		Supported Page Size: 00000553, System Page Size: 00000001
		Region 0: Memory at 58720000 (32-bit, non-prefetchable)
		Region 2: Memory at 58728000 (32-bit, non-prefetchable)
		Region 4: Memory at 58620000 (32-bit, non-prefetchable)
		VF Migration: offset: 00000000, BIR: 0
	Capabilities: [200 v2] Advanced Error Reporting
		UESta:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UEMsk:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UESvrt:	DLP+ SDES- TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
		CESta:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
		CEMsk:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
		AERCap:	First Error Pointer: 00, ECRCGenCap- ECRCGenEn- ECRCChkCap+ ECRCChkEn-
			MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
		HeaderLog: 00000000 00000000 00000000 00000000
	Capabilities: [300 v1] Secondary PCI Express
		LnkCtl3: LnkEquIntrruptEn- PerformEqu-
		LaneErrStat: 0
	Kernel driver in use: pci-endpoint-test
	Kernel modules: pci_endpoint_test

05:00.0 Unassigned class [ff00]: Synopsys, Inc. EPMockUp
	Control: I/O- Mem- BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	Interrupt: pin ? routed to IRQ 145
	IOMMU group: 4
	Region 0: Memory at 58720000 (32-bit, non-prefetchable) [virtual] [size=4K]
	Region 2: Memory at 58728000 (32-bit, non-prefetchable) [virtual] [size=4K]
	Region 4: Memory at 58620000 (32-bit, non-prefetchable) [virtual] [size=128K]
	Capabilities: [80] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 256 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
			ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset- SlotPowerLimit 0.000W
		DevCtl:	CorrErr- NonFatalErr- FatalErr- UnsupReq-
			RlxdOrd- ExtTag- PhantFunc- AuxPwr- NoSnoop-
			MaxPayload 128 bytes, MaxReadReq 128 bytes
		DevSta:	CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
		LnkCap:	Port #1, Speed 8GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <64ns, L1 <1us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
		LnkCtl:	ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed unknown (downgraded), Width x0 (downgraded)
			TrErr- Train- SlotClk- DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+ NROPrPrP- LTR+
			 10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt+ EETLPPrefix-
			 EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
			 FRS- TPHComp- ExtTPHComp-
			 AtomicOpsCap: 32bit- 64bit- 128bitCAS-
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- OBFF Disabled,
			 AtomicOpsCtl: ReqEn-
		LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink- Retimer- 2Retimers- DRS-
		LnkCtl2: Target Link Speed: 2.5GT/s, EnterCompliance- SpeedDis-
			 Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
			 Compliance De-emphasis: -6dB
		LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
			 EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
			 Retimer- 2Retimers- CrosslinkRes: unsupported
	Capabilities: [e0] MSI: Enable+ Count=1/1 Maskable+ 64bit+
		Address: 00000000fffff040  Data: 0000
		Masking: 00000000  Pending: 00000000
	Capabilities: [100 v1] Vendor Specific Information: ID=1556 Rev=1 Len=008 <?>
	Capabilities: [128 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS-, Next Function: 0
		ARICtl:	MFVC- ACS-, Function Group: 0
	Kernel driver in use: pci-endpoint-test
	Kernel modules: pci_endpoint_test
```

### 使用Intel I350网卡测试SR-IOV功能

PF

```bash
04:00.0 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
	Subsystem: Intel Corporation Ethernet Server Adapter I350-T4
	Physical Slot: 0-3
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 32 bytes
	Interrupt: pin A routed to IRQ 73
	IOMMU group: 1
	Region 0: Memory at 58600000 (32-bit, non-prefetchable) [size=1M]
	Region 3: Memory at 58a00000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: [40] Power Management version 3
		Flags: PMEClk- DSI+ D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold+)
		Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=1 PME-
	Capabilities: [50] MSI: Enable- Count=1/1 Maskable+ 64bit+
		Address: 0000000000000000  Data: 0000
		Masking: 00000000  Pending: 00000000
	Capabilities: [70] MSI-X: Enable+ Count=10 Masked-
		Vector table: BAR=3 offset=00000000
		PBA: BAR=3 offset=00002000
	Capabilities: [a0] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 512 bytes, PhantFunc 0, Latency L0s <512ns, L1 <64us
			ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset+ SlotPowerLimit 0.000W
		DevCtl:	CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
			RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+ FLReset-
			MaxPayload 128 bytes, MaxReadReq 512 bytes
		DevSta:	CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr+ TransPend-
		LnkCap:	Port #0, Speed 5GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <4us, L1 <32us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
		LnkCtl:	ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed 5GT/s (ok), Width x1 (downgraded)
			TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+ NROPrPrP- LTR+
			 10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
			 EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
			 FRS- TPHComp- ExtTPHComp-
			 AtomicOpsCap: 32bit- 64bit- 128bitCAS-
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR+ OBFF Disabled,
			 AtomicOpsCtl: ReqEn-
		LnkCtl2: Target Link Speed: 5GT/s, EnterCompliance- SpeedDis-
			 Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
			 Compliance De-emphasis: -6dB
		LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
			 EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
			 Retimer- 2Retimers- CrosslinkRes: unsupported
	Capabilities: [100 v2] Advanced Error Reporting
		UESta:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UEMsk:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UESvrt:	DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
		CESta:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
		CEMsk:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
		AERCap:	First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
			MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
		HeaderLog: 00000000 00000000 00000000 00000000
	Capabilities: [140 v1] Device Serial Number 6c-b3-11-ff-ff-25-d9-c8
	Capabilities: [150 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS-, Next Function: 1
		ARICtl:	MFVC- ACS-, Function Group: 0
	Capabilities: [160 v1] Single Root I/O Virtualization (SR-IOV)
		IOVCap:	Migration-, Interrupt Message Number: 000
		IOVCtl:	Enable+ Migration- Interrupt- MSE+ ARIHierarchy-
		IOVSta:	Migration-
		Initial VFs: 8, Total VFs: 8, Number of VFs: 1, Function Dependency Link: 00
		VF offset: 384, stride: 4, Device ID: 1520
		Supported Page Size: 00000553, System Page Size: 00000001
		Region 0: Memory at 0000001000a00000 (64-bit, prefetchable)
		Region 3: Memory at 0000001000a20000 (64-bit, prefetchable)
		VF Migration: offset: 00000000, BIR: 0
	Capabilities: [1a0 v1] Transaction Processing Hints
		Device specific mode supported
		Steering table in TPH capability structure
	Capabilities: [1c0 v1] Latency Tolerance Reporting
		Max snoop latency: 0ns
		Max no snoop latency: 0ns
	Capabilities: [1d0 v1] Access Control Services
		ACSCap:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
	Kernel driver in use: igb
```

VF

```bash
05:10.0 Ethernet controller: Intel Corporation I350 Ethernet Controller Virtual Function (rev 01)
	Subsystem: Intel Corporation I350 Ethernet Controller Virtual Function
	Control: I/O- Mem- BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	IOMMU group: 8
	Region 0: Memory at 1000a00000 (64-bit, prefetchable) [virtual] [size=16K]
	Region 3: Memory at 1000a20000 (64-bit, prefetchable) [virtual] [size=16K]
	Capabilities: [70] MSI-X: Enable+ Count=3 Masked-
		Vector table: BAR=3 offset=00000000
		PBA: BAR=3 offset=00002000
	Capabilities: [a0] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 512 bytes, PhantFunc 0, Latency L0s <512ns, L1 <64us
			ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset+ SlotPowerLimit 0.000W
		DevCtl:	CorrErr- NonFatalErr- FatalErr- UnsupReq-
			RlxdOrd- ExtTag- PhantFunc- AuxPwr- NoSnoop- FLReset-
			MaxPayload 128 bytes, MaxReadReq 128 bytes
		DevSta:	CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
		LnkCap:	Port #0, Speed 5GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <4us, L1 <32us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
		LnkCtl:	ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed unknown (downgraded), Width x0 (downgraded)
			TrErr- Train- SlotClk- DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+ NROPrPrP- LTR+
			 10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
			 EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
			 FRS- TPHComp- ExtTPHComp-
			 AtomicOpsCap: 32bit- 64bit- 128bitCAS-
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- OBFF Disabled,
			 AtomicOpsCtl: ReqEn-
		LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
			 EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
			 Retimer- 2Retimers- CrosslinkRes: unsupported
	Capabilities: [100 v1] Advanced Error Reporting
		UESta:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UEMsk:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UESvrt:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		CESta:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
		CEMsk:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
		AERCap:	First Error Pointer: 00, ECRCGenCap- ECRCGenEn- ECRCChkCap- ECRCChkEn-
			MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
		HeaderLog: 00000000 00000000 00000000 00000000
	Capabilities: [150 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS-, Next Function: 0
		ARICtl:	MFVC- ACS-, Function Group: 0
	Capabilities: [1a0 v1] Transaction Processing Hints
		Device specific mode supported
		No steering table available
	Capabilities: [1d0 v1] Access Control Services
		ACSCap:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
	Kernel driver in use: igbvf
```

## D3000 EP测试

### 设备树修改

```diff
diff --git a/arch/arm64/boot/dts/phytium/pd2308.dtsi b/arch/arm64/boot/dts/phytium/pd2308.dtsi
index 763c04ac5d30..9107d33433b7 100644
--- a/arch/arm64/boot/dts/phytium/pd2308.dtsi
+++ b/arch/arm64/boot/dts/phytium/pd2308.dtsi
@@ -268,6 +268,7 @@ smmu: iommu@36000000 {
                interrupt-names = "eventq", "priq", "cmdq-sync", "gerror";
                dma-coherent;
                #iommu-cells = <1>;
+               status = "disabled";
        };
 
        soc {
@@ -711,6 +712,7 @@ gdma: gdma@36ce7000 {
                        interrupts = <GIC_SPI 29 IRQ_TYPE_LEVEL_HIGH>,
                                     <GIC_SPI 30 IRQ_TYPE_LEVEL_HIGH>;
                        #dma-cells = <1>;
+                       status = "disabled";
                };
 
                lpc: lpc@20000000 {
@@ -785,5 +787,17 @@ pcie: pcie@40000000 {
                        io-upper = <0x50005000>;
                        dma-coherent;
                };
+
+               ep6: ep@0x340C0000 {
+                       compatible = "phytium,pd2308-pcie-ep";
+                       reg-names = "regs","hpb","mem32","addr_space","atu","dbi2";
+                       reg = <0x0 0x340C0000 0x0 0x8000>,
+                             <0x0 0x34200000 0x0 0x100000>,
+                             <0x0 0x70000000 0x0 0x10000000>,
+                             <0x11 0x00000000 0x01 0x00000000>,
+                             <0x0 0x340D8000 0x0 0x2000>,
+                             <0x0 0x340c8000 0x0 0x2000>;
+                       max-functions = /bits/ 8 <1>;
+               };
        };
 };
```

### 测试过程

测试命令

```bash
#!/bin/bash

modprobe pci-epf-test
mkdir /sys/kernel/config/pci_ep/functions/pci_epf_test/func1
echo 0x16c3 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/vendorid
echo 0xedda > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/deviceid
echo 32 > /sys/kernel/config/pci_ep/functions/pci_epf_test/func1/msi_interrupts
ln -s /sys/kernel/config/pci_ep/functions/pci_epf_test/func1 /sys/kernel/config/pci_ep/controllers/340c0000.ep/
echo 1 > /sys/kernel/config/pci_ep/controllers/340c0000.ep/start
```

EP 设备信息

```bash
root@Ubuntu:/home/jack/Documents/phytium-linux-kernel/tools/pci# lspci -vvv -s 02:00.0
02:00.0 Unassigned class [ff00]: Synopsys, Inc. EPMockUp
	Physical Slot: 0-1
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr+ Stepping- SERR- FastB2B- DisINTx+
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0, Cache Line Size: 32 bytes
	Interrupt: pin A routed to IRQ 105
	Region 0: Memory at 58350400 (32-bit, non-prefetchable) [size=256]
	Region 1: Memory at 58320000 (32-bit, non-prefetchable) [size=64K]
	Region 2: Memory at 58350000 (32-bit, non-prefetchable) [size=1K]
	Region 3: Memory at 58330000 (32-bit, non-prefetchable) [size=64K]
	Region 4: Memory at 58300000 (32-bit, non-prefetchable) [size=128K]
	Region 5: Memory at 58200000 (32-bit, non-prefetchable) [size=1M]
	Expansion ROM at 58340000 [disabled] [size=64K]
	Capabilities: [40] Power Management version 3
		Flags: PMEClk- DSI- D1+ D2+ AuxCurrent=375mA PME(D0+,D1+,D2-,D3hot+,D3cold+)
		Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
	Capabilities: [50] MSI: Enable+ Count=32/32 Maskable- 64bit+
		Address: 00000000fffff040  Data: 0000
	Capabilities: [70] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 512 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
			ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset- SlotPowerLimit 0.000W
		DevCtl:	CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
			RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop-
			MaxPayload 128 bytes, MaxReadReq 512 bytes
		DevSta:	CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr+ TransPend-
		LnkCap:	Port #0, Speed 32GT/s, Width x16, ASPM L1, Exit Latency L1 <64us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
		LnkCtl:	ASPM Disabled; RCB 64 bytes Disabled- CommClk-
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed 8GT/s (downgraded), Width x4 (downgraded)
			TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+, NROPrPrP-, LTR-
			 10BitTagComp+, 10BitTagReq-, OBFF Via WAKE#, ExtFmt-, EETLPPrefix-
			 EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
			 FRS-, TPHComp-, ExtTPHComp-
			 AtomicOpsCap: 32bit- 64bit- 128bitCAS-
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis-, LTR-, OBFF Disabled
			 AtomicOpsCtl: ReqEn-
		LnkCtl2: Target Link Speed: 32GT/s, EnterCompliance- SpeedDis-
			 Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
			 Compliance De-emphasis: -3.5dB
		LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete+, EqualizationPhase1+
			 EqualizationPhase2-, EqualizationPhase3-, LinkEqualizationRequest-
	Capabilities: [100 v2] Advanced Error Reporting
		UESta:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UEMsk:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		UESvrt:	DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
		CESta:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
		CEMsk:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
		AERCap:	First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
			MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
		HeaderLog: 00000000 00000000 00000000 00000000
	Capabilities: [148 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS+, Next Function: 0
		ARICtl:	MFVC- ACS-, Function Group: 0
	Capabilities: [158 v1] Secondary PCI Express
		LnkCtl3: LnkEquIntrruptEn-, PerformEqu-
		LaneErrStat: 0
	Capabilities: [188 v1] Physical Layer 16.0 GT/s <?>
	Capabilities: [1b8 v1] Lane Margining at the Receiver <?>
	Capabilities: [200 v1] Extended Capability ID 0x2a
	Capabilities: [23c v1] L1 PM Substates
		L1SubCap: PCI-PM_L1.2+ PCI-PM_L1.1+ ASPM_L1.2- ASPM_L1.1+ L1_PM_Substates+
			  PortCommonModeRestoreTime=10us PortTPowerOnTime=14us
		L1SubCtl1: PCI-PM_L1.2- PCI-PM_L1.1- ASPM_L1.2- ASPM_L1.1-
			   T_CommonMode=0us
		L1SubCtl2: T_PwrOn=14us
	Capabilities: [24c v1] Vendor Specific Information: ID=0002 Rev=4 Len=100 <?>
	Capabilities: [34c v1] Vendor Specific Information: ID=0001 Rev=1 Len=038 <?>
	Capabilities: [384 v1] Data Link Feature <?>
	Capabilities: [390 v1] Extended Capability ID 0x2f
	Capabilities: [3a0 v1] Designated Vendor-Specific <?>
	Capabilities: [3dc v1] Designated Vendor-Specific <?>
	Capabilities: [410 v1] Designated Vendor-Specific <?>
	Capabilities: [420 v1] Designated Vendor-Specific <?>
	Kernel driver in use: pci-endpoint-test
	Kernel modules: pci_endpoint_test
```

测试结果

```bash
root@Ubuntu:/home/jack/Documents/phytium-linux-kernel/tools/pci# ./pcitest.sh
BAR tests

BAR0:		OKAY
BAR1:		OKAY
BAR2:		OKAY
BAR3:		OKAY
BAR4:		OKAY
BAR5:		OKAY

Interrupt tests

SET IRQ TYPE TO LEGACY:		OKAY
LEGACY IRQ:	NOT OKAY
SET IRQ TYPE TO MSI:		OKAY
MSI1:		OKAY
MSI2:		OKAY
MSI3:		OKAY
MSI4:		OKAY
MSI5:		OKAY
MSI6:		OKAY
MSI7:		OKAY
MSI8:		OKAY
MSI9:		OKAY
MSI10:		OKAY
MSI11:		OKAY
MSI12:		OKAY
MSI13:		OKAY
MSI14:		OKAY
MSI15:		OKAY
MSI16:		OKAY
MSI17:		OKAY
MSI18:		OKAY
MSI19:		OKAY
MSI20:		OKAY
MSI21:		OKAY
MSI22:		OKAY
MSI23:		OKAY
MSI24:		OKAY
MSI25:		OKAY
MSI26:		OKAY
MSI27:		OKAY
MSI28:		OKAY
MSI29:		OKAY
MSI30:		OKAY
MSI31:		OKAY
MSI32:		OKAY

Read Tests

SET IRQ TYPE TO MSI:		OKAY
READ (      1 bytes):		OKAY
READ (   1024 bytes):		OKAY
READ (   1025 bytes):		OKAY
READ (1024000 bytes):		OKAY
READ (1024001 bytes):		OKAY

Write Tests

WRITE (      1 bytes):		OKAY
WRITE (   1024 bytes):		OKAY
WRITE (   1025 bytes):		OKAY
WRITE (1024000 bytes):		OKAY
WRITE (1024001 bytes):		OKAY

Copy Tests

COPY (      1 bytes):		OKAY
COPY (   1024 bytes):		OKAY
COPY (   1025 bytes):		OKAY
COPY (1024000 bytes):		OKAY
COPY (1024001 bytes):		OKAY
```
