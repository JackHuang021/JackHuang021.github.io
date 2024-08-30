---
title: linux CPPC动态调频
date: 2023-09-20 10:49:24
tags:
  - Linux
  - CPPC
categories: Linux
---

### CPPC概述
CPPC(Collaborative Processor Performance Control)协同处理器性能控制，在ACPI规范中描述了CPPC，它是一种操作系统在逻辑处理器的性能范围内管理处理器性能的机制，CPPC使用一组寄存器来描述处理器的性能等级、设置处理器的性能等级、测量处理器的实际性能。

> CPPC在ACPI中的描述位于《ACPI Specification》8.4.6章节  
> CPPC在Linux内核中的文档[cppc_sysfs](https://docs.kernel.org/admin-guide/acpi/cppc_sysfs.html)

OSPM(Operating System-driected Power Management)操作系统定向电源管理，操作系统扮演核心角色，利用操作系统的信息进行电源管理来优化任务的执行。



在CPPC的机制中，platform固件负责创建和维护处理器的性能等级，同时他也可以自主地根据当前的工作负载来控制处理器的性能等级，操作系统OSPM也会向platform固件传递性能一个性能等级目标值来指导platform来进行性能等级的调整，最终调整到哪个性能等级是由platform固件决定的



_CPC(Continuous Performance Control)对象抽象了控制和监控处理器性能的机制，寄存器是由Platform Communication Channel(PCC)接口实现的，下图是CPC定义的给platform固件的指令，这个指令是通过PCC通信机制传递到paltform固件的，这里的指令是对所有核心生效的
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230927144858.png)


S5000C的一个CPC表示例，其中Access Size为Subspace Id，对应到PCCT的子空间，用来和paltform固件通信
```c
Name (CPC6, Package (0x17)
{
	0x17, 
	0x03, 
	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000000, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000004, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000008, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x000000000000000C, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000010, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000014, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000018, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x000000000000001C, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000020, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000024, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x40,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000028, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x40,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000030, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x40,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000038, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000040, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000044, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000048, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x000000000000004C, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000050, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000054, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x0000000000000058, // Address
			0x06,               // Access Size
			)
	}, 

	ResourceTemplate ()
	{
		Register (PCC, 
			0x20,               // Bit Width
			0x00,               // Bit Offset
			0x000000000000005C, // Address
			0x06,               // Access Size
			)
	}
})
```

### CPPC性能控制接口
_CPC performance control package如下：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230921093658.png)
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230921093803.png)

+ Highest Performance: 理想条件下单一核心能够达到的最大性能等级
+ Nominal Performance: 理想条件下处理器能够持续工作的最大性能等级
+ Lowest Nonlinear Performance: 在功耗和效率情况下，能够维持的最小性能等级，此时能确保在维持效率的情况下降低功耗
+ Lowest Performance: 这种情况下，功耗虽然低，但是不能维持效率
+ Guaranteed Performance Register: 表示在当前的外部环境下（电源、温度等），处理器所能维持的最大性能等级，它的值在Lowest Performance和Nominal Performance之间
+ Lowest Frequency and Nominal Frequency: 这两个值由platform提供，表示最低的、标定的CPU频率值，单位MHz，这两个值对应lowest performance和nominal performance，这两个值只是为了确定CPU performance level和频率之间的关系，当操作系统需要上报CPU频率时，这两个值可以用来做推算当前频率

### CPPC的性能控制方法

> 这里所描述到的性能控制方法实际是platform平台固件来执行的，对于OSPM而言，它只向platform固件来传递一个范围在[min performance, max performance]之间的一个连续性能等级给platform固件

OSPM会有一些性能设置选项来影响platform的性能等级设定，OSPM在向platform传递性能值时是采用的platform提供的连续性能值，platform可能会实现一些离散性能等级来对应OSPM传递过来的性能值，这些离散点对应的规则如下：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230921104201.png)

1. OSPM选取的性能等级(Desired Performance)大于等于Guaranteed Performance，那么platform需要在低于guaranteed performance的值，向下取性能等级离散点
2. OSPM选取的性能等级小于guaranteed performance且maximum performance不低于guaranteed performance，那么platform需要向上取性能等级离散点
3. OSPM选取的性能等级和Maximum Performance均均小于Guaranteed Performance，platform需要向上取性能离散点，但是不能超过Maximum Performance

OSPM的一些性能设置寄存器
+ Minimum/Maximum Performance Register: 这两个寄存器主要用来控制OSPM的性能等级范围


+ Desired Performance Register: OSPM会传递该寄存器的性能等级到platform，取值范围在[min performance, max performance]的任意值
+ Performance Reduction Tolerance Register: OSPM表示可允许的与期望性能之间的偏差，platform在控制性能等级时不应该超过该偏差
+ Time Window Register: platform到达期望性能等级的时间窗口，platform在控制性能等级时不应该超过该时间
+ Performance Counters: 根据Refrence Performance Counter和Delivered Performance Counter来计算真实性能水平
+ Reference Performance Counter Register: 参考性能计数，在处理器工作的时候会按照固定速率进行累加
+ Delivered Performance Counter Reigster: 按照当前性能等级进行递增，性能等级越高，增长速率越快，当处理器当前的性能等级为Reference Performance时，它与Reference Performance Counter增长的速率应该是一样的

可以利用Reference Performance Counter、Delivered Performance Counter、Refrence Performance这三个参数来计算当前的实际频率，计算公式如下，代码逻辑在`cppc_cpufreq.c`中的`cppc_cpufreq_get_rate()`中
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230921145254.png)

### PCC通信机制
PCC(Platform Communication Channel)用于OSPM和platform之间的双向通信的标准机制，CPC就是使用PCC和platform进行双向通信的
> PCC的更多描述可以参考《ACPI Specification》14章节  

每个PCC子空间就是一个mailbox通道，PCC实例先在他们自己的表中获取到PCC子空间ID，然后传递给PCC获取mailbox通道来进行通信，具体逻辑在`pcc_mbox_request_channel()`里面。

S5000C中PCCT表中一个子空间的描述
```c
[030h 0048   1]                Subtable Type : 01 [HW-Reduced Comm Subspace]
[031h 0049   1]                       Length : 3E

[032h 0050   4]           Platform Interrupt : 0000001A
[036h 0054   1]        Flags (Decoded Below) : 00
                                    Polarity : 0
                                        Mode : 0
[037h 0055   1]                     Reserved : 00
[038h 0056   8]                 Base Address : 0000000038005800
[040h 0064   8]               Address Length : 0000000000000100

[048h 0072  12]            Doorbell Register : [Generic Address Structure]
[048h 0072   1]                     Space ID : 00 [SystemMemory]
[049h 0073   1]                    Bit Width : 20
[04Ah 0074   1]                   Bit Offset : 00
[04Bh 0075   1]         Encoded Access Width : 03 [DWord Access:32]
[04Ch 0076   8]                      Address : 0000000038001108

[054h 0084   8]                Preserve Mask : 0000000000000000
[05Ch 0092   8]                   Write Mask : 0000000000000001
[064h 0100   4]              Command Latency : 00002710
[068h 0104   4]          Maximum Access Rate : 00000000
[06Ch 0108   2]      Minimum Turnaround Time : 0000
```

根据上面S5000的PCCT表子空间描述，可以知道子空间类型为1（HW-Reduced Communications Subspaces）

HW-Reduced通信子空间的结构如下，对应PCCT表中子空间的描述
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230926171914.png)

PCC共享内存区：为了在platform固件和OSPM之间建立通信，PCC定义了一个邮箱和一个事件接口，邮箱称为“通用通信通道共享内存区域”，platform固件为此会保留一定范围的系统内存。在这个共享内存区域中，为每个PCC实例分配了子空间，给定的PCC实例到底使用哪个子空间，在它们自己相应的ACPI表中指定PCC子空间ID参数，例如CPC表中的Accrss Size就是对应的子空间ID

Doorbell Protocol（门铃协议）：门铃用来OSPM通知platform固件共享内存区域包含有效的准备待处理的命令

门铃包含一个门铃寄存器，门铃寄存器描述在PCC子空间表中，OSPM通过修改这个寄存器指定的位来敲响门铃

OSPM通过PCC子空间向platform固件发送消息的步骤：
1. 首先OSPM通过查看子空间的command complete位来检查子空间是否空闲
2. OSPM将命令放置到子空间的共享内存区域同时更新flags length command payload这些信息，如果PCCT表中描述支持platform中断，则OSPM可以要求platform在命令处理完成后生成一个中断（通过在标志中设置完成时通知位来请求的）
3. platform固件检查命令完成标志位是否清除
4. platform固件处理命令，触发中断，中断描述在PCCT表子空间的GSIV
5. OSPM收到中断设置命令完成标志位

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230926145830.png)


### PSD（P-State Dependency）
这是一个可选的对象，为CPPC性能控制或逻辑处理器的P-State提供一些依赖性的信息，_PSD对象表示一组逻辑处理器之间的性能控制相关属性，PSD在CPPC中实际就是表示CPU的power domain，S5000C 16核处理器是每8个核属于一个power domain
> PSD详细描述可以参考《ACPI Specification》8.4.5.5章节

PSD表详细内容如下
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230927100517.png)

P-state Coordination Types，《ACPI Specification》8.3章节
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230927100932.png)

S5000C的PSD表示例
```c
Name (PSD0, Package (0x01)
{
	Package (0x05)
	{
		0x05, 
		Zero, 
		Zero, 
		0xFD, 	// P-state协作类型属于SW_ANY
		0x08	// 一共有8个核心属于该performance domain
	}
})
```

S5000C的一个CPU节点示例，可以看到该CPU对应PSD0和CPC0
```c
Device (CL00)
{
	Name (_HID, "ACPI0010" /* Processor Container Device */)  // _HID: Hardware ID
	Name (_UID, Zero)  // _UID: Unique ID
	Device (CP00)
	{
		Name (_HID, "ACPI0007" /* Processor Device */)  // _HID: Hardware ID
		Name (_UID, Zero)  // _UID: Unique ID
		Method (_PSD, 0, NotSerialized)  // _PSD: Power State Dependencies
		{
			Return (PSD0) /* \_SB_.PSD0 */
		}

		Method (_CPC, 0, NotSerialized)  // _CPC: Continuous Performance Control
		{
			Return (CPC0) /* \_SB_.CPC0 */
		}
	}

	Device (CP01)
	{
		Name (_HID, "ACPI0007" /* Processor Device */)  // _HID: Hardware ID
		Name (_UID, One)  // _UID: Unique ID
		Method (_PSD, 0, NotSerialized)  // _PSD: Power State Dependencies
		{
			Return (PSD0) /* \_SB_.PSD0 */
		}

		Method (_CPC, 0, NotSerialized)  // _CPC: Continuous Performance Control
		{
			Return (CPC0) /* \_SB_.CPC0 */
		}
	}
}
```

### 代码实现
#### 相关结构体
cppc的实现代码主要位于`drivers/acpi/cppc_acpi.c` `include/acpi/cppc_acpi.h` `drivers/cpufreq/cppc_cpufreq.c`，内核代码基于linux 5.15.125

`struct cppc_cpudata`结构体，policy->driver_data保存的就是该结构体指针

```c
// cpu_data_list用来管理cppc_cpudata
static LIST_HEAD(cpu_data_list);

struct cppc_cpudata {
	// 用来链接到cpu_data_list上
    struct list_head node;
    // 存储CPPC中的一些性能控制相关的寄存器值
    struct cppc_perf_caps perf_caps;
    // perf_ctrls中存储desired perf值，这个值是要报告给platform固件的
    struct cppc_perf_ctrls perf_ctrls;
    struct cppc_perf_fb_ctrs perf_fb_ctrs;
    unsigned int shared_type;
    cpumask_var_t shared_cpu_map;
};
```

`struct cpc_desc`结构体，用来抽象CPC表

```c
struct cpc_desc {
    // 存储CPC表的项目
    int num_entries;
    // CPC版本信息，现在一般是3
    int version;
	// CPU逻辑ID
    int cpu_id;
    int write_cmd_status;
    int write_cmd_id;
    // 存储CPC表寄存器信息
    struct cpc_register_resource cpc_regs[MAX_CPC_REG_ENT];
    // 存储psd信息
    struct acpi_psd_package domain_info;
    struct kobject kobj;
};
```

`struct cppc_pcc_data`结构体用来抽象PCC中的子空间，和固件中的paltform进行通信

```c
// drivers/acpi/cppc_acpi.c
struct cppc_pcc_data {
	struct mbox_chan *pcc_channel;
	void __iomem *pcc_comm_addr;
	bool pcc_channel_acquired;
	unsigned int deadline_us;
	unsigned int pcc_mpar, pcc_mrtt, pcc_nominal;

	bool pending_pcc_write_cmd;	/* Any pending/batched PCC write cmds? */
	bool platform_owns_pcc;		/* Ownership of PCC subspace */
	unsigned int pcc_write_cnt;	/* Running count of PCC write commands */

	/*
	 * Lock to provide controlled access to the PCC channel.
	 *
	 * For performance critical usecases(currently cppc_set_perf)
	 *	We need to take read_lock and check if channel belongs to OSPM
	 * before reading or writing to PCC subspace
	 *	We need to take write_lock before transferring the channel
	 * ownership to the platform via a Doorbell
	 *	This allows us to batch a number of CPPC requests if they happen
	 * to originate in about the same time
	 *
	 * For non-performance critical usecases(init)
	 *	Take write_lock for all purposes which gives exclusive access
	 */
	struct rw_semaphore pcc_lock;

	/* Wait queue for CPUs whose requests were batched */
	wait_queue_head_t pcc_write_wait_q;
	ktime_t last_cmd_cmpl_time;
	ktime_t last_mpar_reset;
	int mpar_count;
	int refcount;
};
```

#### PCC mailbox channel注册过程
```c
// drivers/acpi/cppc_acpi.c
static struct cppc_pcc_data *pcc_data[MAX_PCC_SUBSPACES];

// drivers/mailbox/pcc.c
static struct mbox_chan *pcc_mbox_channels;

// 从pcct表中读取子空间个数，初始化mbox_chan
/**
 * acpi_pcc_probe - Parse the ACPI tree for the PCCT.
 *
 * Return: 0 for Success, else errno.
 */
static int __init acpi_pcc_probe(void)
{
	struct acpi_table_header *pcct_tbl;
	struct acpi_subtable_header *pcct_entry;
	struct acpi_table_pcct *acpi_pcct_tbl;
	struct acpi_subtable_proc proc[ACPI_PCCT_TYPE_RESERVED];
	int count, i, rc;
	acpi_status status = AE_OK;

	/* Search for PCCT */
	status = acpi_get_table(ACPI_SIG_PCCT, 0, &pcct_tbl);

	if (ACPI_FAILURE(status) || !pcct_tbl)
		return -ENODEV;

	/* Set up the subtable handlers */
	for (i = ACPI_PCCT_TYPE_GENERIC_SUBSPACE;
	     i < ACPI_PCCT_TYPE_RESERVED; i++) {
		proc[i].id = i;
		proc[i].count = 0;
		proc[i].handler = parse_pcc_subspace;
	}
	// 从表中获取子空间个数
	count = acpi_table_parse_entries_array(ACPI_SIG_PCCT,
			sizeof(struct acpi_table_pcct), proc,
			ACPI_PCCT_TYPE_RESERVED, MAX_PCC_SUBSPACES);
	if (count <= 0 || count > MAX_PCC_SUBSPACES) {
		if (count < 0)
			pr_warn("Error parsing PCC subspaces from PCCT\n");
		else
			pr_warn("Invalid PCCT: %d PCC subspaces\n", count);

		rc = -EINVAL;
		goto err_put_pcct;
	}
	// pcc_mbox_channels分配内存
	// 为每个子空间分配一个mbox_chan，pcc实例需要用到这个mbox_chan来进行通信
	pcc_mbox_channels = kcalloc(count, sizeof(struct mbox_chan),
				    GFP_KERNEL);
	if (!pcc_mbox_channels) {
		pr_err("Could not allocate space for PCC mbox channels\n");
		rc = -ENOMEM;
		goto err_put_pcct;
	}

	pcc_doorbell_vaddr = kcalloc(count, sizeof(void *), GFP_KERNEL);
	if (!pcc_doorbell_vaddr) {
		rc = -ENOMEM;
		goto err_free_mbox;
	}

	pcc_doorbell_ack_vaddr = kcalloc(count, sizeof(void *), GFP_KERNEL);
	if (!pcc_doorbell_ack_vaddr) {
		rc = -ENOMEM;
		goto err_free_db_vaddr;
	}

	pcc_doorbell_irq = kcalloc(count, sizeof(int), GFP_KERNEL);
	if (!pcc_doorbell_irq) {
		rc = -ENOMEM;
		goto err_free_db_ack_vaddr;
	}

	/* Point to the first PCC subspace entry */
	pcct_entry = (struct acpi_subtable_header *) (
		(unsigned long) pcct_tbl + sizeof(struct acpi_table_pcct));

	acpi_pcct_tbl = (struct acpi_table_pcct *) pcct_tbl;
	if (acpi_pcct_tbl->flags & ACPI_PCCT_DOORBELL)
		pcc_mbox_ctrl.txdone_irq = true;

	for (i = 0; i < count; i++) {
		struct acpi_generic_address *db_reg;
		struct acpi_pcct_subspace *pcct_ss;
		pcc_mbox_channels[i].con_priv = pcct_entry;

		if (pcct_entry->type == ACPI_PCCT_TYPE_HW_REDUCED_SUBSPACE ||
		    pcct_entry->type == ACPI_PCCT_TYPE_HW_REDUCED_SUBSPACE_TYPE2) {
			struct acpi_pcct_hw_reduced *pcct_hrss;

			pcct_hrss = (struct acpi_pcct_hw_reduced *) pcct_entry;

			if (pcc_mbox_ctrl.txdone_irq) {
				rc = pcc_parse_subspace_irq(i, pcct_hrss);
				if (rc < 0)
					goto err;
			}
		}
		pcct_ss = (struct acpi_pcct_subspace *) pcct_entry;

		/* If doorbell is in system memory cache the virt address */
		db_reg = &pcct_ss->doorbell_register;
		if (db_reg->space_id == ACPI_ADR_SPACE_SYSTEM_MEMORY)
			pcc_doorbell_vaddr[i] = acpi_os_ioremap(db_reg->address,
							db_reg->bit_width/8);
		pcct_entry = (struct acpi_subtable_header *)
			((unsigned long) pcct_entry + pcct_entry->length);
	}

	pcc_mbox_ctrl.num_chans = count;

	pr_info("Detected %d PCC Subspaces\n", pcc_mbox_ctrl.num_chans);

	return 0;

err:
	kfree(pcc_doorbell_irq);
err_free_db_ack_vaddr:
	kfree(pcc_doorbell_ack_vaddr);
err_free_db_vaddr:
	kfree(pcc_doorbell_vaddr);
err_free_mbox:
	kfree(pcc_mbox_channels);
err_put_pcct:
	acpi_put_table(pcct_tbl);
	return rc;
}

// drivers/acpi/cppc_acpi.c
// pcc实例传递子空间id来获取一个pcc通道
// 根据PCCT表中子空间的描述完善pcc_data的内容
static int register_pcc_channel(int pcc_ss_idx)
{
	struct acpi_pcct_hw_reduced *cppc_ss;
	u64 usecs_lat;

	if (pcc_ss_idx >= 0) {
		pcc_data[pcc_ss_idx]->pcc_channel =
			pcc_mbox_request_channel(&cppc_mbox_cl,	pcc_ss_idx);

		if (IS_ERR(pcc_data[pcc_ss_idx]->pcc_channel)) {
			pr_err("Failed to find PCC channel for subspace %d\n",
			       pcc_ss_idx);
			return -ENODEV;
		}

		/*
		 * The PCC mailbox controller driver should
		 * have parsed the PCCT (global table of all
		 * PCC channels) and stored pointers to the
		 * subspace communication region in con_priv.
		 */
		// 子空间指针存储在con_priv中
		cppc_ss = (pcc_data[pcc_ss_idx]->pcc_channel)->con_priv;

		if (!cppc_ss) {
			pr_err("No PCC subspace found for %d CPPC\n",
			       pcc_ss_idx);
			return -ENODEV;
		}

		/*
		 * cppc_ss->latency is just a Nominal value. In reality
		 * the remote processor could be much slower to reply.
		 * So add an arbitrary amount of wait on top of Nominal.
		 */
		// S5000C表中的latency为0x2710，即10000微秒
		// min_turnaround_time和max_access_rate都为0
		usecs_lat = NUM_RETRIES * cppc_ss->latency;
		pcc_data[pcc_ss_idx]->deadline_us = usecs_lat;
		pcc_data[pcc_ss_idx]->pcc_mrtt = cppc_ss->min_turnaround_time;
		pcc_data[pcc_ss_idx]->pcc_mpar = cppc_ss->max_access_rate;
		pcc_data[pcc_ss_idx]->pcc_nominal = cppc_ss->latency;

		pcc_data[pcc_ss_idx]->pcc_comm_addr =
			acpi_os_ioremap(cppc_ss->base_address, cppc_ss->length);
		if (!pcc_data[pcc_ss_idx]->pcc_comm_addr) {
			pr_err("Failed to ioremap PCC comm region mem for %d\n",
			       pcc_ss_idx);
			return -ENOMEM;
		}

		/* Set flag so that we don't come here for each CPU. */
		pcc_data[pcc_ss_idx]->pcc_channel_acquired = true;
	}

	return 0;
}

// drivers/mailbox/pcc.c
struct mbox_chan *pcc_mbox_request_channel(struct mbox_client *cl,
		int subspace_id)
{
	struct device *dev = pcc_mbox_ctrl.dev;
	struct mbox_chan *chan;
	unsigned long flags;

	/*
	 * Each PCC Subspace is a Mailbox Channel.
	 * The PCC Clients get their PCC Subspace ID
	 * from their own tables and pass it here.
	 * This returns a pointer to the PCC subspace
	 * for the Client to operate on.
	 */
	// 通过子空间id来获取mbox_chan
	chan = get_pcc_channel(subspace_id);

	if (IS_ERR(chan) || chan->cl) {
		dev_err(dev, "Channel not found for idx: %d\n", subspace_id);
		return ERR_PTR(-EBUSY);
	}
	// mbox_chan的一些初始化工作
	spin_lock_irqsave(&chan->lock, flags);
	chan->msg_free = 0;
	chan->msg_count = 0;
	chan->active_req = NULL;
	chan->cl = cl;
	init_completion(&chan->tx_complete);

	if (chan->txdone_method == TXDONE_BY_POLL && cl->knows_txdone)
		chan->txdone_method = TXDONE_BY_ACK;

	spin_unlock_irqrestore(&chan->lock, flags);
	// 注册中断，门铃协议OSPM向paltform固件发送命令需要用到该中断
	if (pcc_doorbell_irq[subspace_id] > 0) {
		int rc;

		rc = devm_request_irq(dev, pcc_doorbell_irq[subspace_id],
				      pcc_mbox_irq, 0, MBOX_IRQ_NAME, chan);
		if (unlikely(rc)) {
			dev_err(dev, "failed to register PCC interrupt %d\n",
				pcc_doorbell_irq[subspace_id]);
			pcc_mbox_free_channel(chan);
			chan = ERR_PTR(rc);
		}
	}

	return chan;
}
EXPORT_SYMBOL_GPL(pcc_mbox_request_channel);

// drivers/mailbox/pcc.c
static struct mbox_chan *get_pcc_channel(int id)
{
	if (id < 0 || id >= pcc_mbox_ctrl.num_chans)
		return ERR_PTR(-ENOENT);

	return &pcc_mbox_channels[id];
}
```

#### CPPC初始化过程
ACPI _CPC表解析，初始化cpc_desc结构体
```c
// drivers/acpi/processor_driver.c
static struct device_driver acpi_processor_driver = {
	.name = "processor",
	.bus = &cpu_subsys,
	.acpi_match_table = processor_device_ids,
	.probe = acpi_processor_start,
	.remove = acpi_processor_stop,
};

acpi_processor_start();
    __acpi_processor_start();
        acpi_cppc_processor_probe();


/**
 * acpi_cppc_processor_probe - Search for per CPU _CPC objects.
 * @pr: Ptr to acpi_processor containing this CPU's logical ID.
 *
 *	Return: 0 for success or negative value for err.
 */
int acpi_cppc_processor_probe(struct acpi_processor *pr)
{
	struct acpi_buffer output = {ACPI_ALLOCATE_BUFFER, NULL};
	union acpi_object *out_obj, *cpc_obj;
	struct cpc_desc *cpc_ptr;
	struct cpc_reg *gas_t;
	struct device *cpu_dev;
	acpi_handle handle = pr->handle;
	unsigned int num_ent, i, cpc_rev;
	int pcc_subspace_id = -1;
	acpi_status status;
	int ret = -EFAULT;

	/* Parse the ACPI _CPC table for this CPU. */
    // 解析ACPI _CPC表
	status = acpi_evaluate_object_typed(handle, "_CPC", NULL, &output,
			ACPI_TYPE_PACKAGE);
	if (ACPI_FAILURE(status)) {
		ret = -ENODEV;
		goto out_buf_free;
	}

	out_obj = (union acpi_object *) output.pointer;

	cpc_ptr = kzalloc(sizeof(struct cpc_desc), GFP_KERNEL);
	if (!cpc_ptr) {
		ret = -ENOMEM;
		goto out_buf_free;
	}

	/* First entry is NumEntries. */
    // 对应ACPI规范8.4.6.1章节进行解析
	cpc_obj = &out_obj->package.elements[0];
	if (cpc_obj->type == ACPI_TYPE_INTEGER)	{
		num_ent = cpc_obj->integer.value;
		if (num_ent <= 1) {
			pr_debug("Unexpected _CPC NumEntries value (%d) for CPU:%d\n",
				 num_ent, pr->id);
			goto out_free;
		}
	} else {
		pr_debug("Unexpected entry type(%d) for NumEntries\n",
				cpc_obj->type);
		goto out_free;
	}

	/* Second entry should be revision. */
	cpc_obj = &out_obj->package.elements[1];
	if (cpc_obj->type == ACPI_TYPE_INTEGER)	{
		cpc_rev = cpc_obj->integer.value;
	} else {
		pr_debug("Unexpected entry type(%d) for Revision\n",
				cpc_obj->type);
		goto out_free;
	}

	if (cpc_rev < CPPC_V2_REV) {
		pr_debug("Unsupported _CPC Revision (%d) for CPU:%d\n", cpc_rev,
			 pr->id);
		goto out_free;
	}

	/*
	 * Disregard _CPC if the number of entries in the return pachage is not
	 * as expected, but support future revisions being proper supersets of
	 * the v3 and only causing more entries to be returned by _CPC.
	 */
    // 这里进行CPC版本和entry个数的判断，不匹配的话是有异常的
	if ((cpc_rev == CPPC_V2_REV && num_ent != CPPC_V2_NUM_ENT) ||
	    (cpc_rev == CPPC_V3_REV && num_ent != CPPC_V3_NUM_ENT) ||
	    (cpc_rev > CPPC_V3_REV && num_ent <= CPPC_V3_NUM_ENT)) {
		pr_debug("Unexpected number of _CPC return package entries (%d) for CPU:%d\n",
			 num_ent, pr->id);
		goto out_free;
	}
	if (cpc_rev > CPPC_V3_REV) {
		num_ent = CPPC_V3_NUM_ENT;
		cpc_rev = CPPC_V3_REV;
	}
    // 更新到cpc_desc结构体
	cpc_ptr->num_entries = num_ent;
	cpc_ptr->version = cpc_rev;

	/* Iterate through remaining entries in _CPC */
    // 获取cpc表中寄存器的地址
	for (i = 2; i < num_ent; i++) {
		cpc_obj = &out_obj->package.elements[i];

		if (cpc_obj->type == ACPI_TYPE_INTEGER)	{
			cpc_ptr->cpc_regs[i-2].type = ACPI_TYPE_INTEGER;
			cpc_ptr->cpc_regs[i-2].cpc_entry.int_value = cpc_obj->integer.value;
		} else if (cpc_obj->type == ACPI_TYPE_BUFFER) {
			gas_t = (struct cpc_reg *)
				cpc_obj->buffer.pointer;

			/*
			 * The PCC Subspace index is encoded inside
			 * the CPC table entries. The same PCC index
			 * will be used for all the PCC entries,
			 * so extract it only once.
			 */
			// CPC寄存器全部位于platform上，采用PCC通信机制进行读取修改
			if (gas_t->space_id == ACPI_ADR_SPACE_PLATFORM_COMM) {
				if (pcc_subspace_id < 0) {
					// 对应CPC表中entry的Access Size属性，pcc_id用于定位PCC子空间结构数组
					pcc_subspace_id = gas_t->access_width;
					if (pcc_data_alloc(pcc_subspace_id))
						goto out_free;
				} else if (pcc_subspace_id != gas_t->access_width) {
					pr_debug("Mismatched PCC ids.\n");
					goto out_free;
				}
			} else if (gas_t->space_id == ACPI_ADR_SPACE_SYSTEM_MEMORY) {
				if (gas_t->address) {
					void __iomem *addr;

					addr = ioremap(gas_t->address, gas_t->bit_width/8);
					if (!addr)
						goto out_free;
					cpc_ptr->cpc_regs[i-2].sys_mem_vaddr = addr;
				}
			} else {
				if (gas_t->space_id != ACPI_ADR_SPACE_FIXED_HARDWARE || !cpc_ffh_supported()) {
					/* Support only PCC ,SYS MEM and FFH type regs */
					pr_debug("Unsupported register type: %d\n", gas_t->space_id);
					goto out_free;
				}
			}

			cpc_ptr->cpc_regs[i-2].type = ACPI_TYPE_BUFFER;
            // 将寄存器信息拷贝到cpc_desc结构体中
			memcpy(&cpc_ptr->cpc_regs[i-2].cpc_entry.reg, gas_t, sizeof(*gas_t));
		} else {
			pr_debug("Err in entry:%d in CPC table of CPU:%d\n", i, pr->id);
			goto out_free;
		}
	}
	per_cpu(cpu_pcc_subspace_idx, pr->id) = pcc_subspace_id;

	/*
	 * Initialize the remaining cpc_regs as unsupported.
	 * Example: In case FW exposes CPPC v2, the below loop will initialize
	 * LOWEST_FREQ and NOMINAL_FREQ regs as unsupported
	 */
    // 未实现的cpc寄存器值赋值为0
	for (i = num_ent - 2; i < MAX_CPC_REG_ENT; i++) {
		cpc_ptr->cpc_regs[i].type = ACPI_TYPE_INTEGER;
		cpc_ptr->cpc_regs[i].cpc_entry.int_value = 0;
	}


	/* Store CPU Logical ID */
    // 更新CPU逻辑ID
	cpc_ptr->cpu_id = pr->id;

	/* Parse PSD data for this CPU */
    // 解析_PSD表，cpc_desc->domain_info从这里面更新
	ret = acpi_get_psd(cpc_ptr, handle);
	if (ret)
		goto out_free;

	/* Register PCC channel once for all PCC subspace ID. */
	if (pcc_subspace_id >= 0 && !pcc_data[pcc_subspace_id]->pcc_channel_acquired) {
		ret = register_pcc_channel(pcc_subspace_id);
		if (ret)
			goto out_free;

		init_rwsem(&pcc_data[pcc_subspace_id]->pcc_lock);
		init_waitqueue_head(&pcc_data[pcc_subspace_id]->pcc_write_wait_q);
	}

	/* Everything looks okay */
	pr_debug("Parsed CPC struct for CPU: %d\n", pr->id);

	/* Add per logical CPU nodes for reading its feedback counters. */
	cpu_dev = get_cpu_device(pr->id);
	if (!cpu_dev) {
		ret = -EINVAL;
		goto out_free;
	}

	/* Plug PSD data into this CPU's CPC descriptor. */
    // 更新到percpu变量中
	per_cpu(cpc_desc_ptr, pr->id) = cpc_ptr;

	ret = kobject_init_and_add(&cpc_ptr->kobj, &cppc_ktype, &cpu_dev->kobj,
			"acpi_cppc");
	if (ret) {
		per_cpu(cpc_desc_ptr, pr->id) = NULL;
		kobject_put(&cpc_ptr->kobj);
		goto out_free;
	}

	init_freq_invariance_cppc();

	kfree(output.pointer);
	return 0;

out_free:
	/* Free all the mapped sys mem areas for this CPU */
	for (i = 2; i < cpc_ptr->num_entries; i++) {
		void __iomem *addr = cpc_ptr->cpc_regs[i-2].sys_mem_vaddr;

		if (addr)
			iounmap(addr);
	}
	kfree(cpc_ptr);

out_buf_free:
	kfree(output.pointer);
	return ret;
}
EXPORT_SYMBOL_GPL(acpi_cppc_processor_probe);
```

CPPC动态调频驱动还是在linux cpufreq框架下实现的，CPPC作为一个`struct cpufreq_driver`在`cppc_cpufreq_init()`中调用`cpufreq_register_driver(&cppc_cpufreq_driver)`进行初始化
```c
// drivers/cpufreq/cppc_cpufreq.c
// cppc_cpufreq_driver结构体定义
static struct cpufreq_driver cppc_cpufreq_driver = {
	.flags = CPUFREQ_CONST_LOOPS,
	.verify = cppc_verify_policy,
	.target = cppc_cpufreq_set_target,
	.get = cppc_cpufreq_get_rate,
	.init = cppc_cpufreq_cpu_init,
	.exit = cppc_cpufreq_cpu_exit,
	.set_boost = cppc_cpufreq_set_boost,
	.attr = cppc_cpufreq_attr,
	.name = "cppc_cpufreq",
};

// 初始化调用关系
cppc_cpufreq_init();
    cpufreq_register_driver(&cppc_cpufreq_driver);
        subsys_interface_register(&cpufreq_interface);
            cpufreq_add_dev();
                cpufreq_online(cpu);
                    cpufreq_driver->init(policy);
                    cppc_cpufreq_cpu_init();


static int cppc_cpufreq_cpu_init(struct cpufreq_policy *policy)
{
	unsigned int cpu = policy->cpu;
	struct cppc_cpudata *cpu_data;
	struct cppc_perf_caps *caps;
	int ret;

    // acpi_get_psd_map()中更新cpu_data->share_cpu_map的值 
    // cppc_get_perf_caps()读出寄存器的值，更新到cpu_data->perf_caps
    // 调频相关的一些频率值是从这里读出来的
    // 链接cpu_data到cpu_data_list
	// cppc_cpufreq_get_cpu_data()的分析见下
	cpu_data = cppc_cpufreq_get_cpu_data(cpu);
	if (!cpu_data) {
		pr_err("Error in acquiring _CPC/_PSD data for CPU%d.\n", cpu);
		return -ENODEV;
	}
	caps = &cpu_data->perf_caps;
    // driver_data被设置为了cpu_data
	policy->driver_data = cpu_data;

	/*
	 * Set min to lowest nonlinear perf to avoid any efficiency penalty (see
	 * Section 8.4.7.1.1.5 of ACPI 6.1 spec)
	 */
    // 注意这里policy->min设置成了lowest_nonlinear_perf
    // lowest_nonlinear_perf:
    // 在功耗和效率情况下，能够维持的最小性能等级，此时能确保在维持效率的情况下降低功耗
	// cppc_cpufreq_perf_to_khz()通过(low perf, lowfreq), (nomianal perf, nominal freq)
	// 两个点来构成线性函数推算perf对应的频率值
	policy->min = cppc_cpufreq_perf_to_khz(cpu_data,
					       caps->lowest_nonlinear_perf);
	policy->max = cppc_cpufreq_perf_to_khz(cpu_data,
					       caps->nominal_perf);

	/*
	 * Set cpuinfo.min_freq to Lowest to make the full range of performance
	 * available if userspace wants to use any perf between lowest & lowest
	 * nonlinear perf
	 */
    // cpuinfo.min_freq的值要相对policy->min小一些
	policy->cpuinfo.min_freq = cppc_cpufreq_perf_to_khz(cpu_data,
							    caps->lowest_perf);
	policy->cpuinfo.max_freq = cppc_cpufreq_perf_to_khz(cpu_data,
							    caps->nominal_perf);

	policy->transition_delay_us = cppc_cpufreq_get_transition_delay_us(cpu);
	policy->shared_type = cpu_data->shared_type;

	switch (policy->shared_type) {
	case CPUFREQ_SHARED_TYPE_HW:
	case CPUFREQ_SHARED_TYPE_NONE:
		/* Nothing to be done - we'll have a policy for each CPU */
		break;
	case CPUFREQ_SHARED_TYPE_ANY:
		/*
		 * All CPUs in the domain will share a policy and all cpufreq
		 * operations will use a single cppc_cpudata structure stored
		 * in policy->driver_data.
		 */
		// policy中也记录了CPU共享policy的关系，直接把cpu_data中的CPU map复制过去就行
		cpumask_copy(policy->cpus, cpu_data->shared_cpu_map);
		break;
	default:
		pr_debug("Unsupported CPU co-ord type: %d\n",
			 policy->shared_type);
		ret = -EFAULT;
		goto out;
	}

	/*
	 * If 'highest_perf' is greater than 'nominal_perf', we assume CPU Boost
	 * is supported.
	 */
	// 最大性能等级大于连续稳定工作下的性能等级，则置可以超频标志
	// 设置boost超频的时候会用到，将policy->max设置为highest_perf
	if (caps->highest_perf > caps->nominal_perf)
		boost_supported = true;

	/* Set policy->cur to max now. The governors will adjust later. */
	// 初次设置为最大频率，这里记录到了两个位置
	policy->cur = cppc_cpufreq_perf_to_khz(cpu_data, caps->highest_perf);
	cpu_data->perf_ctrls.desired_perf =  caps->highest_perf;
	// 将性能等级设置下去，这里传进去的参数是cpudata->perf_ctrls
	ret = cppc_set_perf(cpu, &cpu_data->perf_ctrls);
	if (ret) {
		pr_debug("Err setting perf value:%d on CPU:%d. ret:%d\n",
			 caps->highest_perf, cpu, ret);
		goto out;
	}

	cppc_cpufreq_cpu_fie_init(policy);
	return 0;

out:
	cppc_cpufreq_put_cpu_data(policy);
	return ret;
}
```

`cppc_cpufreq_get_cpu_data()`，构建cpu和power domain的关系，获取cppc性能参数
```c
// drivers/cpufreq/cppc_cpufreq.c
static struct cppc_cpudata *cppc_cpufreq_get_cpu_data(unsigned int cpu)
{
	struct cppc_cpudata *cpu_data;
	int ret;
	// 为cppc_cpudata结构体分配内存
	// 每个CPU都会维护一份struct cppc_cpudata结构
	cpu_data = kzalloc(sizeof(struct cppc_cpudata), GFP_KERNEL);
	if (!cpu_data)
		goto out;

	if (!zalloc_cpumask_var(&cpu_data->shared_cpu_map, GFP_KERNEL))
		goto free_cpu;
	// power domain对应到CPU的关系
	ret = acpi_get_psd_map(cpu, cpu_data);
	if (ret) {
		pr_debug("Err parsing CPU%d PSD data: ret:%d\n", cpu, ret);
		goto free_mask;
	}
	// 获取cppc定义的性能参数
	ret = cppc_get_perf_caps(cpu, &cpu_data->perf_caps);
	if (ret) {
		pr_debug("Err reading CPU%d perf caps: ret:%d\n", cpu, ret);
		goto free_mask;
	}

	/* Convert the lowest and nominal freq from MHz to KHz */
	cpu_data->perf_caps.lowest_freq *= 1000;
	cpu_data->perf_caps.nominal_freq *= 1000;
	// 链接cpu_data到cpu_data_list
	list_add(&cpu_data->node, &cpu_data_list);

	return cpu_data;

free_mask:
	free_cpumask_var(cpu_data->shared_cpu_map);
free_cpu:
	kfree(cpu_data);
out:
	return NULL;
}

// drivers/acpi/cppc_acpi.c
// 标记对应同一个power domain的CPU
/**
 * acpi_get_psd_map - Map the CPUs in the freq domain of a given cpu
 * @cpu: Find all CPUs that share a domain with cpu.
 * @cpu_data: Pointer to CPU specific CPPC data including PSD info.
 *
 *	Return: 0 for success or negative value for err.
 */
int acpi_get_psd_map(unsigned int cpu, struct cppc_cpudata *cpu_data)
{
	struct cpc_desc *cpc_ptr, *match_cpc_ptr;
	struct acpi_psd_package *match_pdomain;
	struct acpi_psd_package *pdomain;
	int count_target, i;

	/*
	 * Now that we have _PSD data from all CPUs, let's setup P-state
	 * domain info.
	 */
	cpc_ptr = per_cpu(cpc_desc_ptr, cpu);
	if (!cpc_ptr)
		return -EFAULT;

	pdomain = &(cpc_ptr->domain_info);
	cpumask_set_cpu(cpu, cpu_data->shared_cpu_map);
	if (pdomain->num_processors <= 1)
		return 0;

	/* Validate the Domain info */
	count_target = pdomain->num_processors;
	if (pdomain->coord_type == DOMAIN_COORD_TYPE_SW_ALL)
		cpu_data->shared_type = CPUFREQ_SHARED_TYPE_ALL;
	else if (pdomain->coord_type == DOMAIN_COORD_TYPE_HW_ALL)
		cpu_data->shared_type = CPUFREQ_SHARED_TYPE_HW;
	else if (pdomain->coord_type == DOMAIN_COORD_TYPE_SW_ANY)
		cpu_data->shared_type = CPUFREQ_SHARED_TYPE_ANY;
	// 记录在同一个power_domain的CPU
	for_each_possible_cpu(i) {
		if (i == cpu)
			continue;

		match_cpc_ptr = per_cpu(cpc_desc_ptr, i);
		if (!match_cpc_ptr)
			goto err_fault;

		match_pdomain = &(match_cpc_ptr->domain_info);
		if (match_pdomain->domain != pdomain->domain)
			continue;

		/* Here i and cpu are in the same domain */
		if (match_pdomain->num_processors != count_target)
			goto err_fault;

		if (pdomain->coord_type != match_pdomain->coord_type)
			goto err_fault;

		cpumask_set_cpu(i, cpu_data->shared_cpu_map);
	}

	return 0;

err_fault:
	/* Assume no coordination on any error parsing domain info */
	cpumask_clear(cpu_data->shared_cpu_map);
	cpumask_set_cpu(cpu, cpu_data->shared_cpu_map);
	cpu_data->shared_type = CPUFREQ_SHARED_TYPE_NONE;

	return -EFAULT;
}
EXPORT_SYMBOL_GPL(acpi_get_psd_map);


// 读取几个调频相关的参数值
/**
 * cppc_get_perf_caps - Get a CPU's performance capabilities.
 * @cpunum: CPU from which to get capabilities info.
 * @perf_caps: ptr to cppc_perf_caps. See cppc_acpi.h
 *
 * Return: 0 for success with perf_caps populated else -ERRNO.
 */
int cppc_get_perf_caps(int cpunum, struct cppc_perf_caps *perf_caps)
{
	struct cpc_desc *cpc_desc = per_cpu(cpc_desc_ptr, cpunum);
	struct cpc_register_resource *highest_reg, *lowest_reg,
		*lowest_non_linear_reg, *nominal_reg, *guaranteed_reg,
		*low_freq_reg = NULL, *nom_freq_reg = NULL;
	u64 high, low, guaranteed, nom, min_nonlinear, low_f = 0, nom_f = 0;
	int pcc_ss_id = per_cpu(cpu_pcc_subspace_idx, cpunum);
	struct cppc_pcc_data *pcc_ss_data = NULL;
	int ret = 0, regs_in_pcc = 0;

	if (!cpc_desc) {
		pr_debug("No CPC descriptor for CPU:%d\n", cpunum);
		return -ENODEV;
	}
	// 取得对应的寄存器结构体，这些寄存器结构体在 acpi_cppc_processor_probe()中初始化
	highest_reg = &cpc_desc->cpc_regs[HIGHEST_PERF];
	lowest_reg = &cpc_desc->cpc_regs[LOWEST_PERF];
	lowest_non_linear_reg = &cpc_desc->cpc_regs[LOW_NON_LINEAR_PERF];
	nominal_reg = &cpc_desc->cpc_regs[NOMINAL_PERF];
	low_freq_reg = &cpc_desc->cpc_regs[LOWEST_FREQ];
	nom_freq_reg = &cpc_desc->cpc_regs[NOMINAL_FREQ];
	guaranteed_reg = &cpc_desc->cpc_regs[GUARANTEED_PERF];

	/* Are any of the regs PCC ?*/
	if (CPC_IN_PCC(highest_reg) || CPC_IN_PCC(lowest_reg) ||
		CPC_IN_PCC(lowest_non_linear_reg) || CPC_IN_PCC(nominal_reg) ||
		CPC_IN_PCC(low_freq_reg) || CPC_IN_PCC(nom_freq_reg)) {
		if (pcc_ss_id < 0) {
			pr_debug("Invalid pcc_ss_id\n");
			return -ENODEV;
		}
		// 根据子空间id取得子空间数据结构
		pcc_ss_data = pcc_data[pcc_ss_id];
		regs_in_pcc = 1;
		down_write(&pcc_ss_data->pcc_lock);
		/* Ring doorbell once to update PCC subspace */
		// 使用门铃协议向paltform固件发送读数据命令
		if (send_pcc_cmd(pcc_ss_id, CMD_READ) < 0) {
			ret = -EIO;
			goto out_err;
		}
	}

	cpc_read(cpunum, highest_reg, &high);
	perf_caps->highest_perf = high;

	cpc_read(cpunum, lowest_reg, &low);
	perf_caps->lowest_perf = low;

	cpc_read(cpunum, nominal_reg, &nom);
	perf_caps->nominal_perf = nom;

	if (guaranteed_reg->type != ACPI_TYPE_BUFFER  ||
	    IS_NULL_REG(&guaranteed_reg->cpc_entry.reg)) {
		perf_caps->guaranteed_perf = 0;
	} else {
		cpc_read(cpunum, guaranteed_reg, &guaranteed);
		perf_caps->guaranteed_perf = guaranteed;
	}

	cpc_read(cpunum, lowest_non_linear_reg, &min_nonlinear);
	perf_caps->lowest_nonlinear_perf = min_nonlinear;

	if (!high || !low || !nom || !min_nonlinear)
		ret = -EFAULT;

	/* Read optional lowest and nominal frequencies if present */
	if (CPC_SUPPORTED(low_freq_reg))
		cpc_read(cpunum, low_freq_reg, &low_f);

	if (CPC_SUPPORTED(nom_freq_reg))
		cpc_read(cpunum, nom_freq_reg, &nom_f);

	perf_caps->lowest_freq = low_f;
	perf_caps->nominal_freq = nom_f;


out_err:
	if (regs_in_pcc)
		up_write(&pcc_ss_data->pcc_lock);
	return ret;
}
EXPORT_SYMBOL_GPL(cppc_get_perf_caps);
```

#### CPPC频率调整接口
`cppc_set_perf()`解析，这里是将desired perf发送到platform固件，走的是门铃协议

```c
/**
 * cppc_set_perf - Set a CPU's performance controls.
 * @cpu: CPU for which to set performance controls.
 * @perf_ctrls: ptr to cppc_perf_ctrls. See cppc_acpi.h
 *
 * Return: 0 for success, -ERRNO otherwise.
 */
int cppc_set_perf(int cpu, struct cppc_perf_ctrls *perf_ctrls)
{
	struct cpc_desc *cpc_desc = per_cpu(cpc_desc_ptr, cpu);
	struct cpc_register_resource *desired_reg;
	int pcc_ss_id = per_cpu(cpu_pcc_subspace_idx, cpu);
	struct cppc_pcc_data *pcc_ss_data = NULL;
	int ret = 0;

	if (!cpc_desc) {
		pr_debug("No CPC descriptor for CPU:%d\n", cpu);
		return -ENODEV;
	}
	// 先取得desired_reg结构体指针，初始化的时候存在cpc_desc->cpc_regs中
	desired_reg = &cpc_desc->cpc_regs[DESIRED_PERF];

	/*
	 * This is Phase-I where we want to write to CPC registers
	 * -> We want all CPUs to be able to execute this phase in parallel
	 *
	 * Since read_lock can be acquired by multiple CPUs simultaneously we
	 * achieve that goal here
	 */
	// 第一个阶段是获取读者锁，将desired_reg的值更新为perf_ctrls->desired_perf
	// 这个阶段CPU之间是可以并发执行的
	if (CPC_IN_PCC(desired_reg)) {
		if (pcc_ss_id < 0) {
			pr_debug("Invalid pcc_ss_id\n");
			return -ENODEV;
		}
		pcc_ss_data = pcc_data[pcc_ss_id];
		down_read(&pcc_ss_data->pcc_lock); /* BEGIN Phase-I */
		if (pcc_ss_data->platform_owns_pcc) {
			ret = check_pcc_chan(pcc_ss_id, false);
			if (ret) {
				up_read(&pcc_ss_data->pcc_lock);
				return ret;
			}
		}
		/*
		 * Update the pending_write to make sure a PCC CMD_READ will not
		 * arrive and steal the channel during the switch to write lock
		 */
		pcc_ss_data->pending_pcc_write_cmd = true;
		cpc_desc->write_cmd_id = pcc_ss_data->pcc_write_cnt;
		cpc_desc->write_cmd_status = 0;
	}

	/*
	 * Skip writing MIN/MAX until Linux knows how to come up with
	 * useful values.
	 */
	cpc_write(cpu, desired_reg, perf_ctrls->desired_perf);

	if (CPC_IN_PCC(desired_reg))
		up_read(&pcc_ss_data->pcc_lock);	/* END Phase-I */
	/*
	 * This is Phase-II where we transfer the ownership of PCC to Platform
	 *
	 * Short Summary: Basically if we think of a group of cppc_set_perf
	 * requests that happened in short overlapping interval. The last CPU to
	 * come out of Phase-I will enter Phase-II and ring the doorbell.
	 *
	 * We have the following requirements for Phase-II:
	 *     1. We want to execute Phase-II only when there are no CPUs
	 * currently executing in Phase-I
	 *     2. Once we start Phase-II we want to avoid all other CPUs from
	 * entering Phase-I.
	 *     3. We want only one CPU among all those who went through Phase-I
	 * to run phase-II
	 *
	 * If write_trylock fails to get the lock and doesn't transfer the
	 * PCC ownership to the platform, then one of the following will be TRUE
	 *     1. There is at-least one CPU in Phase-I which will later execute
	 * write_trylock, so the CPUs in Phase-I will be responsible for
	 * executing the Phase-II.
	 *     2. Some other CPU has beaten this CPU to successfully execute the
	 * write_trylock and has already acquired the write_lock. We know for a
	 * fact it (other CPU acquiring the write_lock) couldn't have happened
	 * before this CPU's Phase-I as we held the read_lock.
	 *     3. Some other CPU executing pcc CMD_READ has stolen the
	 * down_write, in which case, send_pcc_cmd will check for pending
	 * CMD_WRITE commands by checking the pending_pcc_write_cmd.
	 * So this CPU can be certain that its request will be delivered
	 *    So in all cases, this CPU knows that its request will be delivered
	 * by another CPU and can return
	 *
	 * After getting the down_write we still need to check for
	 * pending_pcc_write_cmd to take care of the following scenario
	 *    The thread running this code could be scheduled out between
	 * Phase-I and Phase-II. Before it is scheduled back on, another CPU
	 * could have delivered the request to Platform by triggering the
	 * doorbell and transferred the ownership of PCC to platform. So this
	 * avoids triggering an unnecessary doorbell and more importantly before
	 * triggering the doorbell it makes sure that the PCC channel ownership
	 * is still with OSPM.
	 *   pending_pcc_write_cmd can also be cleared by a different CPU, if
	 * there was a pcc CMD_READ waiting on down_write and it steals the lock
	 * before the pcc CMD_WRITE is completed. send_pcc_cmd checks for this
	 * case during a CMD_READ and if there are pending writes it delivers
	 * the write command before servicing the read command
	 */
	// 第二个阶段通过PCC子空间传递写命令，这个阶段只允许一个CPU进行
	// 根据APCI规范，platform固件只要收到了命令会处理所有的寄存器
	if (CPC_IN_PCC(desired_reg)) {
		if (down_write_trylock(&pcc_ss_data->pcc_lock)) {/* BEGIN Phase-II */
			/* Update only if there are pending write commands */
			if (pcc_ss_data->pending_pcc_write_cmd)
				send_pcc_cmd(pcc_ss_id, CMD_WRITE);
			up_write(&pcc_ss_data->pcc_lock);	/* END Phase-II */
		} else
			/* Wait until pcc_write_cnt is updated by send_pcc_cmd */
			// pcc_write_cnt在每次写命令时会递增，所以当write_cmd_id和pcc_write_cnt
			// 不相等时则可以判断写命令已经完成了
			wait_event(pcc_ss_data->pcc_write_wait_q,
				   cpc_desc->write_cmd_id != pcc_ss_data->pcc_write_cnt);

		/* send_pcc_cmd updates the status in case of failure */
		ret = cpc_desc->write_cmd_status;
	}
	return ret;
}
EXPORT_SYMBOL_GPL(cppc_set_perf);
```

#### CPPC频率获取接口
CPPC的频率获取是要根据Reference Performance Counter Register和Delivered Performance Counter Register来计算的，
```c
static unsigned int cppc_cpufreq_get_rate(unsigned int cpu)
{
	struct cppc_perf_fb_ctrs fb_ctrs_t0 = {0}, fb_ctrs_t1 = {0};
	struct cpufreq_policy *policy = cpufreq_cpu_get(cpu);
	struct cppc_cpudata *cpu_data = policy->driver_data;
	u64 delivered_perf;
	int ret;

	cpufreq_cpu_put(policy);
	// 先读取一下两个counter的值和refrence performance
	ret = cppc_get_perf_ctrs(cpu, &fb_ctrs_t0);
	if (ret)
		return ret;

	udelay(2); /* 2usec delay between sampling */
	// 经过固定时间再读取一遍
	ret = cppc_get_perf_ctrs(cpu, &fb_ctrs_t1);
	if (ret)
		return ret;
	// 根据ACPI规范 8.4.6.1.3 Performance Feedback章节计算当前实际频率
	delivered_perf = cppc_perf_from_fbctrs(cpu_data, &fb_ctrs_t0,
					       &fb_ctrs_t1);

	return cppc_cpufreq_perf_to_khz(cpu_data, delivered_perf);
}
```

### S5000C cpufreq-info信息

```bash
# cpufreq-info 
cpufrequtils 008: cpufreq-info (C) Dominik Brodowski 2004-2009
Report errors and bugs to cpufreq@vger.kernel.org, please.
analyzing CPU 0:
  driver: cppc_cpufreq
  # 8个核属于同一个power domain
  CPUs which run at the same hardware frequency: 0 1 4 5 8 9 12 13
  CPUs which need to have their frequency coordinated by software: 0 1 4 5 8 9 12 13
  maximum transition latency: 4294.55 ms.
  # S5000C没有频率表的概念，CPPC采用连续频率值进行调频，范围为[50MHz, 2.1GHz]
  hardware limits: 50.0 MHz - 2.10 GHz
  available cpufreq governors: conservative, ondemand, userspace, powersave, performance, schedutil
  current policy: frequency should be within 50.0 MHz and 2.10 GHz.
                  The governor "ondemand" may decide which speed to use
                  within this range.
  current CPU frequency is 500 MHz (asserted by call to hardware).
```



> 参考
> 1. https://www.cnblogs.com/lvzh/p/17061923.html
> 2. https://blog.csdn.net/tiantao2012/article/details/87917968
> 3. [linux kernel documents cppc_sysfs.txt](https://docs.kernel.org/admin-guide/acpi/cppc_sysfs.html)
> 4. https://blog.csdn.net/qq_21186033/article/details/117015605
> 5. https://blog.csdn.net/qq_21186033/article/details/117015409

