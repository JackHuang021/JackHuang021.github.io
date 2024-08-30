---
title: Linux MMC驱动框架
tags:
  - Linux
  - MMC
categories:
  - Linux
abbrlink: a0a95d0f
date: 2023-01-29 14:37:36
---

### 1. Linux MMC驱动子系统
块设备是Linux中的基础外设之一，而MMC/SD存储设备是一种典型的块设备，Linux内核设计了MMC（Muilti Media Card）子系统，用于管理MMC/SD设备

硬件特性：卡与主控制器间串行传送，工作时钟频率范围为0~200MHz

使用MMC接口规范（Multimedia Card Interface）的设备都可以称为MMC设备，MMC设备的种类：
1. mmc type card
2. sd type card
3. sdio type card 

<!-- more -->

MMC驱动子系统包含三个部分：
1. bus: MMC总线(mmc_bus)
2. host: 封装在platform_device下的host设备
3. card: 抽象具体的MMC卡，有对应的mmc driver


core layer根据MMC/SD协议标准实现了协议，card layer与Linux的块设备子系统对接，实现块设备驱动以及完成请求，具体协议经过core layer的接口，最终通过host layer完成传输，对MMC设备进行实际的操作

host和card可以理解为MMC device的MMC主设备和MMC从设备，其中host为集成于芯片内部的MMC controller，card为MMC设备内部实际的存储设备

### 2. MMC协议

#### 2.1 MMC命令
SDIO总线上的设置和控制都是通过命令来实现的，SDIO总线上都是HOST端发起请求，然后DEVICE端回应请求，其中请求和应答中会包含数据信息：
+ **Command：** 用于开始传输的命令，是由HOST端发往DEVICE端的，其中命令是通过CMD信号线传送的
+ **Response：** DEVICE返回的应答，也是通过CMD信号线传送的
+ **Data：** 数据是双向传送的，可以设置为1线模式，也可以设置为4线模式，数据是通过DAT0-DAT3信号线传输的

##### 2.1.1 命令格式
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230129145056.png)
+ Start Bit：起始位，固定为0
+ Transmission：传输方向，值为1表示由host发出，0则表示由device发出
+ Command Index：代表命令索引，例如CMD5的值为5，范围为0-63
+ Argument：CMD所附带的一些参数，不同的CMD，这32bit每一位所代表的含义是不一样的
+ CRC7：7位CRC校验值
+ END：结束位，固定为1


### 3. SD协议
参考 《SD Specifications Part 1 Physical Layer Simplified Specification Version 9.10 December 1,2023》

#### 3.1 卡状态和操作模式
主机控制通信过程，主机会发送两种类型的命令：广播命令、寻址命令

SD协议中定义了两种操作模式：
1. 卡识别模式：在复位后，主机会处于卡识别模式，卡在复位后会处于识别模式，直到收到SEND_RCA(CMD3)命令
2. 数据传输模式：当RCA第一次发布后，卡会处于数据传输模式；主机会在总线上所有的卡都被识别后进入这个模式

卡操作模式和状态的关系
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240822101251.png)

#### 3.2 SD卡寄存器介绍
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240821164147.png)

SD卡内部有7个寄存器
1. OCR、CID、CSD、SCR寄存器保存卡的配置信息
2. CSR寄存器和SSR寄存器保存着卡的状态信息

##### 3.2.1 Card Identification Register(CID)
CID寄存器长度为128 bits，包含了卡的生产信息，主控制器不能修改其内容
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240822085542.png)

##### 3.2.2 Card Specific Data(CSD)
CSD寄存器长度为128 bits，包含了访问该卡数据时的一些配置信息
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240822090159.png)

##### 3.2.3 SD Card Configuration Register(SCR)
SCR寄存器长度为64 bits，存储了一些SD卡的特性信息
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240821170937.png)


##### 3.2.4 Operating Conditions Register(OCR)
OCR寄存器长度为32 bits，存储了卡支持的操作电压范围，
bit15 - bit23是支持的电压范围
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240814171418.png)

##### 3.2.5 Relative Card Address(RCA)
RCA寄存器的长度为16 bits，存储了卡的通信地址，这个寄存器只有在MMC总线模式下才有效


#### 3.3 SD卡规格识别
1. 早期的速度等级使用Class等级来标识，规定了最低的连续写入速度，Class等级越高传输速度越快，现在比较常见的都是Class 10级别，可简称C10，最低连续写入速度为10MB/s
2. UHS（Ultra High Speed）超高速等级为全新的总线模式。目前有UHS-I、UHS-II、UHS-III三种版本（向下兼容），可区分最大读取速度，同样会在卡面标识。每种版本的UHS标准，又使用U1和U3的单数等级标识最低写入速度，以便跟之前Class等级的双数等级进行区分，卡面上以字母U中间加数字作为标识。
3. VSC（Video Speed Class）视频速度等级。随着4K的逐渐普及，SD协会针对视频拍摄应用制定了新的视频速度等级。以识别字母V加上最低写入速度值的数字构成标识。
4. 最后，一个新分类随着 Android 的 Adopted Storage Device 功能的引入而推出。App Performance Class 确保最低随机和顺序性能速度可满足特定条件下的运行和存储执行时间要求。App Performance Class 有两个级别，分别是 A1 和 A2。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240815194735.png)

各种速度等级对应的最低写入速度表格
![](https://raw.githubusercontent.com/JackHuang021/images/master/SD_speed.png)

从一张闪迪的SD卡来说明卡面的标识
![](https://raw.githubusercontent.com/JackHuang021/images/master/SD卡-fotor-2024081519425.png)

1. 卡品牌
2. 超高速卡标识（Ultra High Speed）
3. 卡容量为32GB
4. micro SD卡标识，表示这张卡为micro SD卡
5. 表示这张卡支持U1的速度等级，最低连续写入速度为10MB/s
6. 这张卡为SDHC容量的卡，SDHC的容量等级为2-32GB
7. 表示这张卡支持UHS-I模式，最大读取速度为104MB/s
8. 早期的速度标识，这张卡支持Class10，最低连续写入速度为10MB/s
9. A1为App Performance Class，其规定了最低随机读取、写入的IOPS，一共有两个级别，A1, A2

总结一下这张卡标识所给出的信息：
1. 容量32GB
2. 最低连续写入速度为10MB/s，支持UHS-I模式，最大连续读取速度为104MB/s


### 4. MMC驱动框架
Linux内核中把mmc、sd以及sdio三者的驱动代码整合在一起，俗称MMC子系统，源码位于drivers/mmc目录下，mmc目录下有core和host两个文件夹
+ host：针对不同主机端的SDIO、MMC控制器的驱动，这部分需要由驱动工程师来完成
+ core：整个MMC的核心层，这部分实现了不通协议和规范，为HOST层和设备驱动层提供接口函数，还存放了块设备的相关驱动

Linux内核中，使用两个结构体`struct mmc_host`和`struct mmc_card`分别描述host和card，其中host设备被封装成platform_device注册到Linux驱动模型中，整体而言，Linux驱动模型框架下，MMC驱动子系统包括三个部分：
+ MMC总线（mmc_bus）
+ 封装在`platform_device`下的host设备
+ 依附于MMC总线的MMC驱动（mmc_driver）
#### 4.1 mmc class与bus注册
```c
// drivers/mmc/core/bus.c
// mmc bus定义
static struct bus_type mmc_bus_type = {
	.name = "mmc",
	.dev_groups = mmc_dev_groups,
	.match 		= mmc_bus_match,
	.uevent		= mmc_bus_uevent,
	.probe 		= mmc_bus_probe,
	.remove 	= mmc_bus_remove,
	.shutdown 	= mmc_bus_shutdown,
	.pm			= &mmc_bus_pm_ops,
};
// mmc bus注册
int mmc_register_bus(void)
{
	return bus_register(&mmc_bus_type);
}


// drivers/mmc/core/host.c
// mmc host class定义
static struct class mmc_host_class = {
	.name		= "mmc_host",
	.dev_release	= mmc_host_classdev_release,
	.shutdown_pre	= mmc_host_classdev_shutdown,
	.pm		= MMC_HOST_CLASS_DEV_PM_OPS,
};
// mmc host class注册
int mmc_register_host_class(void)
{
	return class_register(&mmc_host_class);
}

// drivers/mmc/core/sdio_bus.c
// sdio bus定义
static struct bus_type sdio_bus_type = {
	.name		= "sdio",
	.dev_groups	= sdio_dev_groups,
	.match		= sdio_bus_match,
	.uevent		= sdio_bus_uevent,
	.probe		= sdio_bus_probe,
	.remove		= sdio_bus_remove,
	.pm		= &sdio_bus_pm_ops,
};
// sdio bus注册
int sdio_register_bus(void)
{
	return bus_register(&sdio_bus_type);
}

// drivers/mmc/core/core.c
// mmc core初始化
static int __init mmc_init(void)
{
	int ret;

	// mmc总线注册，对应sysfs下的/sys/bus/mmc/目录
	ret = mmc_register_bus();
	if (ret)
		return ret;

	// mmc_host class注册，对应sysfs下的/sys/class/mmc_host目录
	ret = mmc_register_host_class();
	if (ret)
		goto unregister_bus;

	ret = sdio_register_bus();
	if (ret)
		goto unregister_host_class;

	return 0;

unregister_host_class:
	mmc_unregister_host_class();
unregister_bus:
	mmc_unregister_bus();
	return ret;
}
```

主要包括两个方面：
+ 利用`bus_register()`注册mmc_bus，对应sysfs下的`/sys/bus/mmc/`目录
+ 利用`class_register()`注册mmc_host_class，对应`sysfs下的/sys/class/mmc_host`目录

#### 4.2 host相关结构体
MMC设备主要包括主设备host和从设备card两部分，而主设备host将被封装在platform_device中注册到驱动模型中。

`struct mmc_host`结构体
```c
// include/linux/mmc/host.h
// mmc_host结构体定义
struct mmc_host {
	struct device		*parent;
	struct device		class_dev;
	int			index;
	const struct mmc_host_ops *ops;
	struct mmc_pwrseq	*pwrseq;
	unsigned int		f_min;
	unsigned int		f_max;
	unsigned int		f_init;
	u32			ocr_avail;
	u32			ocr_avail_sdio;	/* SDIO-specific OCR */
	u32			ocr_avail_sd;	/* SD-specific OCR */
	u32			ocr_avail_mmc;	/* MMC-specific OCR */
#ifdef CONFIG_PM_SLEEP
	struct notifier_block	pm_notify;
#endif
	struct wakeup_source	*ws;		/* Enable consume of uevents */
	u32			max_current_330;
	u32			max_current_300;
	u32			max_current_180;

#define MMC_VDD_165_195		0x00000080	/* VDD voltage 1.65 - 1.95 */
#define MMC_VDD_20_21		0x00000100	/* VDD voltage 2.0 ~ 2.1 */
#define MMC_VDD_21_22		0x00000200	/* VDD voltage 2.1 ~ 2.2 */
#define MMC_VDD_22_23		0x00000400	/* VDD voltage 2.2 ~ 2.3 */
#define MMC_VDD_23_24		0x00000800	/* VDD voltage 2.3 ~ 2.4 */
#define MMC_VDD_24_25		0x00001000	/* VDD voltage 2.4 ~ 2.5 */
#define MMC_VDD_25_26		0x00002000	/* VDD voltage 2.5 ~ 2.6 */
#define MMC_VDD_26_27		0x00004000	/* VDD voltage 2.6 ~ 2.7 */
#define MMC_VDD_27_28		0x00008000	/* VDD voltage 2.7 ~ 2.8 */
#define MMC_VDD_28_29		0x00010000	/* VDD voltage 2.8 ~ 2.9 */
#define MMC_VDD_29_30		0x00020000	/* VDD voltage 2.9 ~ 3.0 */
#define MMC_VDD_30_31		0x00040000	/* VDD voltage 3.0 ~ 3.1 */
#define MMC_VDD_31_32		0x00080000	/* VDD voltage 3.1 ~ 3.2 */
#define MMC_VDD_32_33		0x00100000	/* VDD voltage 3.2 ~ 3.3 */
#define MMC_VDD_33_34		0x00200000	/* VDD voltage 3.3 ~ 3.4 */
#define MMC_VDD_34_35		0x00400000	/* VDD voltage 3.4 ~ 3.5 */
#define MMC_VDD_35_36		0x00800000	/* VDD voltage 3.5 ~ 3.6 */

	u32			caps;		/* Host capabilities */

#define MMC_CAP_4_BIT_DATA	(1 << 0)	/* Can the host do 4 bit transfers */
#define MMC_CAP_MMC_HIGHSPEED	(1 << 1)	/* Can do MMC high-speed timing */
#define MMC_CAP_SD_HIGHSPEED	(1 << 2)	/* Can do SD high-speed timing */
#define MMC_CAP_SDIO_IRQ	(1 << 3)	/* Can signal pending SDIO IRQs */
#define MMC_CAP_SPI		(1 << 4)	/* Talks only SPI protocols */
#define MMC_CAP_NEEDS_POLL	(1 << 5)	/* Needs polling for card-detection */
#define MMC_CAP_8_BIT_DATA	(1 << 6)	/* Can the host do 8 bit transfers */
#define MMC_CAP_AGGRESSIVE_PM	(1 << 7)	/* Suspend (e)MMC/SD at idle  */
#define MMC_CAP_NONREMOVABLE	(1 << 8)	/* Nonremovable e.g. eMMC */
#define MMC_CAP_WAIT_WHILE_BUSY	(1 << 9)	/* Waits while card is busy */
#define MMC_CAP_3_3V_DDR	(1 << 11)	/* Host supports eMMC DDR 3.3V */
#define MMC_CAP_1_8V_DDR	(1 << 12)	/* Host supports eMMC DDR 1.8V */
#define MMC_CAP_1_2V_DDR	(1 << 13)	/* Host supports eMMC DDR 1.2V */
#define MMC_CAP_DDR		(MMC_CAP_3_3V_DDR | MMC_CAP_1_8V_DDR | \
				 MMC_CAP_1_2V_DDR)
#define MMC_CAP_POWER_OFF_CARD	(1 << 14)	/* Can power off after boot */
#define MMC_CAP_BUS_WIDTH_TEST	(1 << 15)	/* CMD14/CMD19 bus width ok */
#define MMC_CAP_UHS_SDR12	(1 << 16)	/* Host supports UHS SDR12 mode */
#define MMC_CAP_UHS_SDR25	(1 << 17)	/* Host supports UHS SDR25 mode */
#define MMC_CAP_UHS_SDR50	(1 << 18)	/* Host supports UHS SDR50 mode */
#define MMC_CAP_UHS_SDR104	(1 << 19)	/* Host supports UHS SDR104 mode */
#define MMC_CAP_UHS_DDR50	(1 << 20)	/* Host supports UHS DDR50 mode */
#define MMC_CAP_UHS		(MMC_CAP_UHS_SDR12 | MMC_CAP_UHS_SDR25 | \
				 MMC_CAP_UHS_SDR50 | MMC_CAP_UHS_SDR104 | \
				 MMC_CAP_UHS_DDR50)
#define MMC_CAP_SYNC_RUNTIME_PM	(1 << 21)	/* Synced runtime PM suspends. */
#define MMC_CAP_NEED_RSP_BUSY	(1 << 22)	/* Commands with R1B can't use R1. */
#define MMC_CAP_DRIVER_TYPE_A	(1 << 23)	/* Host supports Driver Type A */
#define MMC_CAP_DRIVER_TYPE_C	(1 << 24)	/* Host supports Driver Type C */
#define MMC_CAP_DRIVER_TYPE_D	(1 << 25)	/* Host supports Driver Type D */
#define MMC_CAP_DONE_COMPLETE	(1 << 27)	/* RW reqs can be completed within mmc_request_done() */
#define MMC_CAP_CD_WAKE		(1 << 28)	/* Enable card detect wake */
#define MMC_CAP_CMD_DURING_TFR	(1 << 29)	/* Commands during data transfer */
#define MMC_CAP_CMD23		(1 << 30)	/* CMD23 supported. */
#define MMC_CAP_HW_RESET	(1 << 31)	/* Reset the eMMC card via RST_n */

	u32			caps2;		/* More host capabilities */

#define MMC_CAP2_BOOTPART_NOACC	(1 << 0)	/* Boot partition no access */
#define MMC_CAP2_FULL_PWR_CYCLE	(1 << 2)	/* Can do full power cycle */
#define MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND (1 << 3) /* Can do full power cycle in suspend */
#define MMC_CAP2_HS200_1_8V_SDR	(1 << 5)        /* can support */
#define MMC_CAP2_HS200_1_2V_SDR	(1 << 6)        /* can support */
#define MMC_CAP2_HS200		(MMC_CAP2_HS200_1_8V_SDR | \
				 MMC_CAP2_HS200_1_2V_SDR)
#define MMC_CAP2_CD_ACTIVE_HIGH	(1 << 10)	/* Card-detect signal active high */
#define MMC_CAP2_RO_ACTIVE_HIGH	(1 << 11)	/* Write-protect signal active high */
#define MMC_CAP2_NO_PRESCAN_POWERUP (1 << 14)	/* Don't power up before scan */
#define MMC_CAP2_HS400_1_8V	(1 << 15)	/* Can support HS400 1.8V */
#define MMC_CAP2_HS400_1_2V	(1 << 16)	/* Can support HS400 1.2V */
#define MMC_CAP2_HS400		(MMC_CAP2_HS400_1_8V | \
				 MMC_CAP2_HS400_1_2V)
#define MMC_CAP2_HSX00_1_8V	(MMC_CAP2_HS200_1_8V_SDR | MMC_CAP2_HS400_1_8V)
#define MMC_CAP2_HSX00_1_2V	(MMC_CAP2_HS200_1_2V_SDR | MMC_CAP2_HS400_1_2V)
#define MMC_CAP2_SDIO_IRQ_NOTHREAD (1 << 17)
#define MMC_CAP2_NO_WRITE_PROTECT (1 << 18)	/* No physical write protect pin, assume that card is always read-write */
#define MMC_CAP2_NO_SDIO	(1 << 19)	/* Do not send SDIO commands during initialization */
#define MMC_CAP2_HS400_ES	(1 << 20)	/* Host supports enhanced strobe */
#define MMC_CAP2_NO_SD		(1 << 21)	/* Do not send SD commands during initialization */
#define MMC_CAP2_NO_MMC		(1 << 22)	/* Do not send (e)MMC commands during initialization */
#define MMC_CAP2_CQE		(1 << 23)	/* Has eMMC command queue engine */
#define MMC_CAP2_CQE_DCMD	(1 << 24)	/* CQE can issue a direct command */
#define MMC_CAP2_AVOID_3_3V	(1 << 25)	/* Host must negotiate down from 3.3V */
#define MMC_CAP2_MERGE_CAPABLE	(1 << 26)	/* Host can merge a segment over the segment size */

	int			fixed_drv_type;	/* fixed driver type for non-removable media */

	mmc_pm_flag_t		pm_caps;	/* supported pm features */

	/* host specific block data */
	unsigned int		max_seg_size;	/* see blk_queue_max_segment_size */
	unsigned short		max_segs;	/* see blk_queue_max_segments */
	unsigned short		unused;
	unsigned int		max_req_size;	/* maximum number of bytes in one req */
	unsigned int		max_blk_size;	/* maximum size of one mmc block */
	unsigned int		max_blk_count;	/* maximum number of blocks in one req */
	unsigned int		max_busy_timeout; /* max busy timeout in ms */

	/* private data */
	spinlock_t		lock;		/* lock for claim and bus ops */

	struct mmc_ios		ios;		/* current io bus settings */

	/* group bitfields together to minimize padding */
	unsigned int		use_spi_crc:1;
	// mmc host是否被占用
	unsigned int		claimed:1;	/* host exclusively claimed */
	unsigned int		bus_dead:1;	/* bus has been released */
	unsigned int		doing_init_tune:1; /* initial tuning in progress */
	unsigned int		can_retune:1;	/* re-tuning can be used */
	unsigned int		doing_retune:1;	/* re-tuning in progress */
	unsigned int		retune_now:1;	/* do re-tuning at next req */
	unsigned int		retune_paused:1; /* re-tuning is temporarily disabled */
	unsigned int		use_blk_mq:1;	/* use blk-mq */
	unsigned int		retune_crc_disable:1; /* don't trigger retune upon crc */
	unsigned int		can_dma_map_merge:1; /* merging can be used */

	int			rescan_disable;	/* disable card detection */
	/* 记录是否已经进行了scan */
	int			rescan_entered;	/* used with nonremovable devices */

	int			need_retune;	/* re-tuning is needed */
	int			hold_retune;	/* hold off re-tuning */
	unsigned int		retune_period;	/* re-tuning period in secs */
	struct timer_list	retune_timer;	/* for periodic re-tuning */

	bool			trigger_card_event; /* card_event necessary */

	struct mmc_card		*card;		/* device attached to this host */

	wait_queue_head_t	wq;
	struct mmc_ctx		*claimer;	/* context that has host claimed */
	/* mmc host占用计数 */
	int			claim_cnt;	/* "claim" nesting count */
	struct mmc_ctx		default_ctx;	/* default context */

	struct delayed_work	detect;
	/* 检测到卡有插拔的标志 */
	int			detect_change;	/* card detect flag */
	struct mmc_slot		slot;

	const struct mmc_bus_ops *bus_ops;	/* current bus driver */
	unsigned int		bus_refs;	/* reference counter */

	unsigned int		sdio_irqs;
	struct task_struct	*sdio_irq_thread;
	struct delayed_work	sdio_irq_work;
	bool			sdio_irq_pending;
	atomic_t		sdio_irq_thread_abort;

	mmc_pm_flag_t		pm_flags;	/* requested pm features */

	struct led_trigger	*led;		/* activity led */

#ifdef CONFIG_REGULATOR
	bool			regulator_enabled; /* regulator state */
#endif
	struct mmc_supply	supply;

	struct dentry		*debugfs_root;

	/* Ongoing data transfer that allows commands during transfer */
	/* 记录上一个传输请求 */
	struct mmc_request	*ongoing_mrq;

#ifdef CONFIG_FAIL_MMC_REQUEST
	struct fault_attr	fail_mmc_request;
#endif

	unsigned int		actual_clock;	/* Actual HC clock rate */

	unsigned int		slotno;	/* used for sdio acpi binding */

	int			dsr_req;	/* DSR value is valid */
	u32			dsr;	/* optional driver stage (DSR) value */

	/* Command Queue Engine (CQE) support */
	const struct mmc_cqe_ops *cqe_ops;
	void			*cqe_private;
	int			cqe_qdepth;
	bool			cqe_enabled;
	bool			cqe_on;

	/* Host Software Queue support */
	bool			hsq_enabled;

	unsigned long		private[] ____cacheline_aligned;
};
```

`struct mmc_ios`结构体，存储io bus setting相关，host电源状态
```c
struct mmc_ios {
	unsigned int	clock;			/* clock rate */
	/* mmc controller支持的最小电压 */
	unsigned short	vdd;
	unsigned int	power_delay_ms;		/* waiting for stable power */

/* vdd stores the bit number of the selected voltage range from below. */

	unsigned char	bus_mode;		/* command output mode */

#define MMC_BUSMODE_OPENDRAIN	1
#define MMC_BUSMODE_PUSHPULL	2
	// 使用spi总线连接情况下的片选信号状态
	unsigned char	chip_select;		/* SPI chip select */

#define MMC_CS_DONTCARE		0
#define MMC_CS_HIGH		1
#define MMC_CS_LOW		2

	unsigned char	power_mode;		/* power supply mode */

#define MMC_POWER_OFF		0
#define MMC_POWER_UP		1
#define MMC_POWER_ON		2
#define MMC_POWER_UNDEFINED	3

	unsigned char	bus_width;		/* data bus width */

#define MMC_BUS_WIDTH_1		0
#define MMC_BUS_WIDTH_4		2
#define MMC_BUS_WIDTH_8		3

	unsigned char	timing;			/* timing specification used */

#define MMC_TIMING_LEGACY	0
#define MMC_TIMING_MMC_HS	1
#define MMC_TIMING_SD_HS	2
#define MMC_TIMING_UHS_SDR12	3
#define MMC_TIMING_UHS_SDR25	4
#define MMC_TIMING_UHS_SDR50	5
#define MMC_TIMING_UHS_SDR104	6
#define MMC_TIMING_UHS_DDR50	7
#define MMC_TIMING_MMC_DDR52	8
#define MMC_TIMING_MMC_HS200	9
#define MMC_TIMING_MMC_HS400	10
#define MMC_TIMING_SD_EXP	11
#define MMC_TIMING_SD_EXP_1_2V	12

	unsigned char	signal_voltage;		/* signalling voltage (1.8V or 3.3V) */

#define MMC_SIGNAL_VOLTAGE_330	0
#define MMC_SIGNAL_VOLTAGE_180	1
#define MMC_SIGNAL_VOLTAGE_120	2

	unsigned char	drv_type;		/* driver type (A, B, C, D) */

#define MMC_SET_DRIVER_TYPE_B	0
#define MMC_SET_DRIVER_TYPE_A	1
#define MMC_SET_DRIVER_TYPE_C	2
#define MMC_SET_DRIVER_TYPE_D	3

	bool enhanced_strobe;			/* hs400es selection */
};
```

`struct mmc_command`，对应MMC协议中的命令
```c
struct mmc_command {
	/* 命令索引，对应协议里面的command index，范围为0-63 */
	u32			opcode;
	/* 命令携带的参数，对应协议里面的argument */
	u32			arg;
#define MMC_CMD23_ARG_REL_WR	(1 << 31)
#define MMC_CMD23_ARG_PACKED	((0 << 31) | (1 << 30))
#define MMC_CMD23_ARG_TAG_REQ	(1 << 29)
	/* 存放response数据，最大长度是136 bits */
	u32			resp[4];
	unsigned int		flags;		/* expected response type */
#define MMC_RSP_PRESENT	(1 << 0)
#define MMC_RSP_136	(1 << 1)		/* 136 bit response */
#define MMC_RSP_CRC	(1 << 2)		/* expect valid crc */
#define MMC_RSP_BUSY	(1 << 3)		/* card may send busy */
#define MMC_RSP_OPCODE	(1 << 4)		/* response contains opcode */

#define MMC_CMD_MASK	(3 << 5)		/* non-SPI command type */
#define MMC_CMD_AC	(0 << 5)
#define MMC_CMD_ADTC	(1 << 5)
#define MMC_CMD_BC	(2 << 5)
#define MMC_CMD_BCR	(3 << 5)

#define MMC_RSP_SPI_S1	(1 << 7)		/* one status byte */
#define MMC_RSP_SPI_S2	(1 << 8)		/* second byte */
#define MMC_RSP_SPI_B4	(1 << 9)		/* four data bytes */
#define MMC_RSP_SPI_BUSY (1 << 10)		/* card may send busy */

/*
 * These are the native response types, and correspond to valid bit
 * patterns of the above flags.  One additional valid pattern
 * is all zeros, which means we don't expect a response.
 */
#define MMC_RSP_NONE	(0)
#define MMC_RSP_R1	(MMC_RSP_PRESENT|MMC_RSP_CRC|MMC_RSP_OPCODE)
#define MMC_RSP_R1B	(MMC_RSP_PRESENT|MMC_RSP_CRC|MMC_RSP_OPCODE|MMC_RSP_BUSY)
#define MMC_RSP_R2	(MMC_RSP_PRESENT|MMC_RSP_136|MMC_RSP_CRC)
#define MMC_RSP_R3	(MMC_RSP_PRESENT)
#define MMC_RSP_R4	(MMC_RSP_PRESENT)
#define MMC_RSP_R5	(MMC_RSP_PRESENT|MMC_RSP_CRC|MMC_RSP_OPCODE)
#define MMC_RSP_R6	(MMC_RSP_PRESENT|MMC_RSP_CRC|MMC_RSP_OPCODE)
#define MMC_RSP_R7	(MMC_RSP_PRESENT|MMC_RSP_CRC|MMC_RSP_OPCODE)

/* Can be used by core to poll after switch to MMC HS mode */
#define MMC_RSP_R1_NO_CRC	(MMC_RSP_PRESENT|MMC_RSP_OPCODE)

#define mmc_resp_type(cmd)	((cmd)->flags & (MMC_RSP_PRESENT|MMC_RSP_136|MMC_RSP_CRC|MMC_RSP_BUSY|MMC_RSP_OPCODE))

/*
 * These are the SPI response types for MMC, SD, and SDIO cards.
 * Commands return R1, with maybe more info.  Zero is an error type;
 * callers must always provide the appropriate MMC_RSP_SPI_Rx flags.
 */
#define MMC_RSP_SPI_R1	(MMC_RSP_SPI_S1)
#define MMC_RSP_SPI_R1B	(MMC_RSP_SPI_S1|MMC_RSP_SPI_BUSY)
#define MMC_RSP_SPI_R2	(MMC_RSP_SPI_S1|MMC_RSP_SPI_S2)
#define MMC_RSP_SPI_R3	(MMC_RSP_SPI_S1|MMC_RSP_SPI_B4)
#define MMC_RSP_SPI_R4	(MMC_RSP_SPI_S1|MMC_RSP_SPI_B4)
#define MMC_RSP_SPI_R5	(MMC_RSP_SPI_S1|MMC_RSP_SPI_S2)
#define MMC_RSP_SPI_R7	(MMC_RSP_SPI_S1|MMC_RSP_SPI_B4)

#define mmc_spi_resp_type(cmd)	((cmd)->flags & \
		(MMC_RSP_SPI_S1|MMC_RSP_SPI_BUSY|MMC_RSP_SPI_S2|MMC_RSP_SPI_B4))

/*
 * These are the command types.
 */
#define mmc_cmd_type(cmd)	((cmd)->flags & MMC_CMD_MASK)
	/* 命令重试次数 */
	unsigned int		retries;	/* max number of retries */
	int			error;		/* command error */

/*
 * Standard errno values are used for errors, but some have specific
 * meaning in the MMC layer:
 *
 * ETIMEDOUT    Card took too long to respond
 * EILSEQ       Basic format problem with the received or sent data
 *              (e.g. CRC check failed, incorrect opcode in response
 *              or bad end bit)
 * EINVAL       Request cannot be performed because of restrictions
 *              in hardware and/or the driver
 * ENOMEDIUM    Host can determine that the slot is empty and is
 *              actively failing requests
 */

	unsigned int		busy_timeout;	/* busy detect timeout in ms */
	struct mmc_data		*data;		/* data segment associated with cmd */
	struct mmc_request	*mrq;		/* associated request */
};
```

`struct mmc_request`结构体
```c
struct mmc_request {
	struct mmc_command	*sbc;		/* SET_BLOCK_COUNT for multiblock */
	struct mmc_command	*cmd;
	struct mmc_data		*data;
	struct mmc_command	*stop;

	struct completion	completion;
	struct completion	cmd_completion;
	void			(*done)(struct mmc_request *);/* completion function */
	/*
	 * Notify uppers layers (e.g. mmc block driver) that recovery is needed
	 * due to an error associated with the mmc_request. Currently used only
	 * by CQE.
	 */
	void			(*recovery_notifier)(struct mmc_request *);
	struct mmc_host		*host;

	/* Allow other commands during this ongoing data transfer or busy wait */
	bool			cap_cmd_during_tfr;

	int			tag;

#ifdef CONFIG_MMC_CRYPTO
	const struct bio_crypt_ctx *crypto_ctx;
	int			crypto_key_slot;
#endif
};
```

#### 4.3 card相关的结构体
`struct mmc_card`
```c
/*
 * MMC device
 */
struct mmc_card {
	struct mmc_host		*host;		/* the host this device belongs to */
	struct device		dev;		/* the device */
	/* 存储ocr的值，这里存放的ocr是经过和卡电压匹配过的 */
	u32			ocr;		/* the current OCR setting */
	unsigned int		rca;		/* relative card address of device */
	unsigned int		type;		/* card type */
#define MMC_TYPE_MMC		0		/* MMC card */
#define MMC_TYPE_SD		1		/* SD card */
#define MMC_TYPE_SDIO		2		/* SDIO card */
#define MMC_TYPE_SD_COMBO	3		/* SD combo (IO+mem) card */
	unsigned int		state;		/* (our) card state */
	unsigned int		quirks; 	/* card quirks */
	unsigned int		quirk_max_rate;	/* max rate set by quirks */
#define MMC_QUIRK_LENIENT_FN0	(1<<0)		/* allow SDIO FN0 writes outside of the VS CCCR range */
#define MMC_QUIRK_BLKSZ_FOR_BYTE_MODE (1<<1)	/* use func->cur_blksize */
						/* for byte mode */
#define MMC_QUIRK_NONSTD_SDIO	(1<<2)		/* non-standard SDIO card attached */
						/* (missing CIA registers) */
#define MMC_QUIRK_NONSTD_FUNC_IF (1<<4)		/* SDIO card has nonstd function interfaces */
#define MMC_QUIRK_DISABLE_CD	(1<<5)		/* disconnect CD/DAT[3] resistor */
#define MMC_QUIRK_INAND_CMD38	(1<<6)		/* iNAND devices have broken CMD38 */
#define MMC_QUIRK_BLK_NO_CMD23	(1<<7)		/* Avoid CMD23 for regular multiblock */
#define MMC_QUIRK_BROKEN_BYTE_MODE_512 (1<<8)	/* Avoid sending 512 bytes in */
						/* byte mode */
#define MMC_QUIRK_LONG_READ_TIME (1<<9)		/* Data read time > CSD says */
#define MMC_QUIRK_SEC_ERASE_TRIM_BROKEN (1<<10)	/* Skip secure for erase/trim */
#define MMC_QUIRK_BROKEN_IRQ_POLLING	(1<<11)	/* Polling SDIO_CCCR_INTx could create a fake interrupt */
#define MMC_QUIRK_TRIM_BROKEN	(1<<12)		/* Skip trim */
#define MMC_QUIRK_BROKEN_HPI	(1<<13)		/* Disable broken HPI support */
#define MMC_QUIRK_BROKEN_SD_DISCARD	(1<<14)	/* Disable broken SD discard support */
#define MMC_QUIRK_BROKEN_SD_CACHE	(1<<15)	/* Disable broken SD cache support */
#define MMC_QUIRK_BROKEN_CACHE_FLUSH	(1<<16)	/* Don't flush cache until the write has occurred */

	bool			written_flag;	/* Indicates eMMC has been written since power on */
	bool			reenable_cmdq;	/* Re-enable Command Queue */

	unsigned int		erase_size;	/* erase size in sectors */
 	unsigned int		erase_shift;	/* if erase unit is power 2 */
 	unsigned int		pref_erase;	/* in sectors */
	unsigned int		eg_boundary;	/* don't cross erase-group boundaries */
	unsigned int		erase_arg;	/* erase / trim / discard */
 	u8			erased_byte;	/* value of erased bytes */
	/* 存储卡原始的cid寄存器值 */
	u32			raw_cid[4];	/* raw card CID */
	/* 存储卡原始的csd寄存器值 */
	u32			raw_csd[4];	/* raw card CSD */
	u32			raw_scr[2];	/* raw card SCR */
	u32			raw_ssr[16];	/* raw card SSR */
	struct mmc_cid		cid;		/* card identification */
	struct mmc_csd		csd;		/* card specific */
	struct mmc_ext_csd	ext_csd;	/* mmc v4 extended card specific */
	/* 存放SCR寄存器信息 */
	struct sd_scr		scr;		/* extra SD information */
	struct sd_ssr		ssr;		/* yet more SD information */
	struct sd_switch_caps	sw_caps;	/* switch (CMD6) caps */
	struct sd_ext_reg	ext_power;	/* SD extension reg for PM */
	struct sd_ext_reg	ext_perf;	/* SD extension reg for PERF */

	unsigned int		sdio_funcs;	/* number of SDIO functions */
	atomic_t		sdio_funcs_probed; /* number of probed SDIO funcs */
	struct sdio_cccr	cccr;		/* common card info */
	struct sdio_cis		cis;		/* common tuple info */
	struct sdio_func	*sdio_func[SDIO_MAX_FUNCS]; /* SDIO functions (devices) */
	struct sdio_func	*sdio_single_irq; /* SDIO function when only one IRQ active */
	u8			major_rev;	/* major revision number */
	u8			minor_rev;	/* minor revision number */
	unsigned		num_info;	/* number of info strings */
	const char		**info;		/* info strings */
	struct sdio_func_tuple	*tuples;	/* unknown common tuples */

	unsigned int		sd_bus_speed;	/* Bus Speed Mode set for the card */
	unsigned int		mmc_avail_type;	/* supported device type by both host and card */
	unsigned int		drive_strength;	/* for UHS-I, HS200 or HS400 */

	struct dentry		*debugfs_root;
	struct mmc_part	part[MMC_NUM_PHY_PARTITION]; /* physical partitions */
	unsigned int    nr_parts;

	struct workqueue_struct *complete_wq;	/* Private workqueue */
};
```

cid和csd的数据结构
```c
struct mmc_cid {
	unsigned int		manfid;
	char			prod_name[8];
	unsigned char		prv;
	unsigned int		serial;
	unsigned short		oemid;
	unsigned short		year;
	unsigned char		hwrev;
	unsigned char		fwrev;
	unsigned char		month;
};

struct mmc_csd {
	unsigned char		structure;
	unsigned char		mmca_vsn;
	unsigned short		cmdclass;
	unsigned short		taac_clks;
	unsigned int		taac_ns;
	unsigned int		c_size;
	unsigned int		r2w_factor;
	unsigned int		max_dtr;
	unsigned int		erase_size;		/* In sectors */
	unsigned int		read_blkbits;
	unsigned int		write_blkbits;
	unsigned int		capacity;
	unsigned int		read_partial:1,
				read_misalign:1,
				write_partial:1,
				write_misalign:1,
				dsr_imp:1;
};
```

#### 4.4 sys目录下文件节点说明
/sys目录下文件节点的说明，mmc bus节点对应的路径为/sys/bus/mmc，在`mmc_register_bus()`中生成
```bash
# /sys/bus/mmc/devices/ 目录下对应的是mmc core抽象出来的card设备
# 这两个card设备就对应E2000 demo板上的SD卡和板载emmc
root@Ubuntu:~# ls /sys/bus/mmc/devices/
mmc0:0001  mmc1:59b4

# /sys/bus/mmc/drivers/ 目录下对应的是card driver
# 其中mmcblk就是block.c中实现的card driver
root@Ubuntu:~# ls /sys/bus/mmc/drivers
mmcblk
```

mmc host的class节点：mmc core实现了一个class用于维护和管理mmc host，其对应的路径为`/sys/class/mmc_host`，mmc core会为每个注册到mmc core中的mmc host在该class目录下添加一个对应的节点
```bash
# mmc0和mmc1分别对应E2000上的两个mmc controller
root@Ubuntu:/sys/class/mmc_host# ls
mmc0  mmc1
```

### 5. phytium mmc host驱动

#### 5.1 E2000Q demo 板设备树描述
```c
mmc@28000000 {
		compatible = "phytium,mci";
		reg = <0x00000000 0x28000000 0x00000000 0x00001000>;
		interrupts = <0x00000000 0x00000048 0x00000004>;
		clocks = <0x0000000b>;
		clock-names = "phytium_mci_clk";
		status = "okay";
		bus-width = <0x00000008>;
		max-frequency = <0x05f5e100>;
		cap-mmc-hw-reset;
		cap-mmc-highspeed;
		mmc-hs200-1_8v;
		no-sdio;
		no-sd;
		non-removable;
		phandle = <0x0000001b>;
};
mmc@28001000 {
		compatible = "phytium,mci";
		reg = <0x00000000 0x28001000 0x00000000 0x00001000>;
		interrupts = <0x00000000 0x00000049 0x00000004>;
		clocks = <0x0000000b>;
		clock-names = "phytium_mci_clk";
		status = "okay";
		bus-width = <0x00000004>;
		max-frequency = <0x02faf080>;
		cap-sdio-irq;
		cap-sd-highspeed;
		no-sdio;
		no-mmc;
		phandle = <0x0000001c>;
};
```

### 6. SD卡测试

```bash
读：fio -filename=/dev/mmcblk0 -direct=1 -numjobs=8 -thread -group_reporting -ioengine=libaio -iodepth=1 -size=1G -name=test -bs=1M -rw=read
写：fio -filename=/dev/mmcblk0 -direct=1 -numjobs=8 -thread -group_reporting -ioengine=libaio -iodepth=1 -size=1G -name=test -bs=1M -rw=write
```

#### 6.1 50M uhs-sdr-104

SD卡信息
```bash
root@Ubuntu:/sys/kernel/debug/mmc0# cat ios
clock:		50000000 Hz
vdd:		21 (3.3 ~ 3.4 V)
bus mode:	2 (push-pull)
chip select:	0 (don't care)
power mode:	2 (on)
bus width:	2 (4 bits)
timing spec:	6 (sd uhs SDR104)
signal voltage:	1 (1.80 V)
driver type:	0 (driver type B)
```

读取速度
```bash
root@Ubuntu:/sys/kernel/debug/mmc0# fio -filename=/dev/mmcblk0 -direct=1 -numjobs=8 -thread -group_reporting -ioengine=libaio -iodepth=1 -size=1G -name=test -bs=1M -rw=read
test: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.16
Starting 8 threads
Jobs: 2 (f=1): [f(1),_(5),R(1),_(1)][98.4%][r=23.0MiB/s][r=23 IOPS][eta 00m:06s]          
test: (groupid=0, jobs=8): err= 0: pid=1495: Tue Aug 20 15:32:30 2024
  read: IOPS=22, BW=22.5MiB/s (23.6MB/s)(8192MiB/363725msec)
    slat (usec): min=65, max=1642, avg=119.64, stdev=44.77
    clat (msec): min=43, max=5719, avg=353.49, stdev=168.04
     lat (msec): min=44, max=5719, avg=353.61, stdev=168.04
    clat percentiles (msec):
     |  1.00th=[  132],  5.00th=[  264], 10.00th=[  309], 20.00th=[  330],
     | 30.00th=[  351], 40.00th=[  351], 50.00th=[  351], 60.00th=[  351],
     | 70.00th=[  351], 80.00th=[  372], 90.00th=[  393], 95.00th=[  439],
     | 99.00th=[  527], 99.50th=[  567], 99.90th=[  676], 99.95th=[ 5403],
     | 99.99th=[ 5738]
   bw (  KiB/s): min=15852, max=56014, per=100.00%, avg=23483.00, stdev=1052.48, samples=5696
   iops        : min=    8, max=   54, avg=22.83, stdev= 1.03, samples=5696
  lat (msec)   : 50=0.22%, 100=0.48%, 250=3.37%, 500=94.42%, 750=1.42%
  lat (msec)   : >=2000=0.10%
  cpu          : usr=0.00%, sys=0.03%, ctx=8215, majf=0, minf=2056
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=8192,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=22.5MiB/s (23.6MB/s), 22.5MiB/s-22.5MiB/s (23.6MB/s-23.6MB/s), io=8192MiB (8590MB), run=363725-363725msec
```

写入速度
```bash
root@Ubuntu:/sys/kernel/debug/mmc0# fio -filename=/dev/mmcblk0 -direct=1 -numjobs=8 -thread -group_reporting -ioengine=libaio -iodepth=1 -size=200M -name=test -bs=1M -rw=write
test: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.16
Starting 8 threads
Jobs: 1 (f=1): [_(5),W(1),_(2)][97.9%][w=11.0MiB/s][w=11 IOPS][eta 00m:04s]          
test: (groupid=0, jobs=8): err= 0: pid=1580: Tue Aug 20 15:54:15 2024
  write: IOPS=8, BW=8607KiB/s (8813kB/s)(1600MiB/190364msec); 0 zone resets
    slat (usec): min=44, max=259, avg=139.62, stdev=50.05
    clat (msec): min=89, max=2023, avg=933.78, stdev=301.00
     lat (msec): min=89, max=2023, avg=933.92, stdev=301.00
    clat percentiles (msec):
     |  1.00th=[   92],  5.00th=[  447], 10.00th=[  550], 20.00th=[  726],
     | 30.00th=[  743], 40.00th=[  852], 50.00th=[  986], 60.00th=[ 1045],
     | 70.00th=[ 1099], 80.00th=[ 1183], 90.00th=[ 1301], 95.00th=[ 1368],
     | 99.00th=[ 1519], 99.50th=[ 1670], 99.90th=[ 1838], 99.95th=[ 2022],
     | 99.99th=[ 2022]
   bw (  KiB/s): min=15036, max=32173, per=100.00%, avg=16738.02, stdev=420.23, samples=1551
   iops        : min=    8, max=   31, avg=16.06, stdev= 0.41, samples=1551
  lat (msec)   : 100=1.69%, 250=0.75%, 500=6.00%, 750=22.00%, 1000=20.94%
  lat (msec)   : 2000=48.56%, >=2000=0.06%
  cpu          : usr=0.01%, sys=0.01%, ctx=1642, majf=0, minf=8
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1600,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=8607KiB/s (8813kB/s), 8607KiB/s-8607KiB/s (8813kB/s-8813kB/s), io=1600MiB (1678MB), run=190364-190364msec

Disk stats (read/write):
  mmcblk0: ios=134/3195, merge=0/0, ticks=59715/2556916, in_queue=2616632, util=100.00%
```

#### 6.2 100M uhs-sdr-104模式

测试环境： D3000 demo A主板，uboot固件，kernel v6.6

SD卡信息
```bash
root@Ubuntu:/sys/kernel/debug/mmc0# cat ios
clock:		100000000 Hz
vdd:		21 (3.3 ~ 3.4 V)
bus mode:	2 (push-pull)
chip select:	0 (don't care)
power mode:	2 (on)
bus width:	2 (4 bits)
timing spec:	6 (sd uhs SDR104)
signal voltage:	1 (1.80 V)
driver type:	0 (driver type B)
```

连续读速度
```bash
root@Ubuntu:/sys/kernel/debug/mmc0# fio -filename=/dev/mmcblk0 -direct=1 -numjobs=8 -thread -group_reporting -ioengine=libaio -iodepth=1 -size=1G -name=test -bs=1M -rw=read
test: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.16
Starting 8 threads
Jobs: 8 (f=8): [R(8)][100.0%][r=45.0MiB/s][r=45 IOPS][eta 00m:00s]
test: (groupid=0, jobs=8): err= 0: pid=942: Tue Aug 20 16:07:18 2024
  read: IOPS=44, BW=44.7MiB/s (46.8MB/s)(8192MiB/183414msec)
    slat (usec): min=71, max=1591, avg=116.46, stdev=41.97
    clat (msec): min=35, max=269, avg=178.91, stdev=52.93
     lat (msec): min=36, max=270, avg=179.03, stdev=52.93
    clat percentiles (msec):
     |  1.00th=[   67],  5.00th=[   90], 10.00th=[  112], 20.00th=[  134],
     | 30.00th=[  157], 40.00th=[  157], 50.00th=[  180], 60.00th=[  203],
     | 70.00th=[  224], 80.00th=[  224], 90.00th=[  247], 95.00th=[  271],
     | 99.00th=[  271], 99.50th=[  271], 99.90th=[  271], 99.95th=[  271],
     | 99.99th=[  271]
   bw (  KiB/s): min=28618, max=65536, per=99.98%, avg=45724.91, stdev=1222.57, samples=2928
   iops        : min=   22, max=   64, avg=43.99, stdev= 1.22, samples=2928
  lat (msec)   : 50=0.02%, 100=8.33%, 250=83.30%, 500=8.35%
  cpu          : usr=0.00%, sys=0.07%, ctx=8214, majf=0, minf=2056
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=8192,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=44.7MiB/s (46.8MB/s), 44.7MiB/s-44.7MiB/s (46.8MB/s-46.8MB/s), io=8192MiB (8590MB), run=183414-183414msec

Disk stats (read/write):
  mmcblk0: ios=16383/0, merge=0/0, ticks=2503325/0, in_queue=2503325, util=100.00%
```

读取速度
```bash
root@Ubuntu:/sys/kernel/debug/mmc0# fio -filename=/dev/mmcblk0 -direct=1 -numjobs=8 -thread -group_reporting -ioengine=libaio -iodepth=1 -size=200M -name=test -bs=1M -rw=write
test: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=1
...
fio-3.16
Starting 8 threads
Jobs: 7 (f=7): [W(1),f(1),W(5),_(1)][99.3%][w=9225KiB/s][w=9 IOPS][eta 00m:01s]
test: (groupid=0, jobs=8): err= 0: pid=1033: Tue Aug 20 16:21:32 2024
  write: IOPS=11, BW=11.7MiB/s (12.2MB/s)(1600MiB/137332msec); 0 zone resets
    slat (usec): min=39, max=251, avg=116.72, stdev=50.22
    clat (msec): min=85, max=1579, avg=685.23, stdev=283.54
     lat (msec): min=85, max=1579, avg=685.34, stdev=283.54
    clat percentiles (msec):
     |  1.00th=[   86],  5.00th=[  268], 10.00th=[  342], 20.00th=[  439],
     | 30.00th=[  518], 40.00th=[  617], 50.00th=[  667], 60.00th=[  726],
     | 70.00th=[  793], 80.00th=[  911], 90.00th=[ 1099], 95.00th=[ 1200],
     | 99.00th=[ 1452], 99.50th=[ 1485], 99.90th=[ 1569], 99.95th=[ 1586],
     | 99.99th=[ 1586]
   bw (  KiB/s): min=15428, max=32768, per=100.00%, avg=17494.51, stdev=507.11, samples=1491
   iops        : min=    8, max=   32, avg=16.75, stdev= 0.52, samples=1491
  lat (msec)   : 100=1.38%, 250=2.44%, 500=24.75%, 750=36.94%, 1000=18.75%
  lat (msec)   : 2000=15.75%
  cpu          : usr=0.01%, sys=0.01%, ctx=1611, majf=0, minf=8
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1600,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=11.7MiB/s (12.2MB/s), 11.7MiB/s-11.7MiB/s (12.2MB/s-12.2MB/s), io=1600MiB (1678MB), run=137332-137332msec

Disk stats (read/write):
  mmcblk0: ios=146/3191, merge=0/0, ticks=39355/1808944, in_queue=1848299, util=100.00%
```







