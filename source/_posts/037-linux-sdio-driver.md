---
title: Linux MMC子系统
date: 2023-01-29 14:37:36
tags:
    - Linux
    - MMC
categories:
    - Linux
---

### Linux MMC驱动子系统
块设备是Linux中的基础外设之一，而MMC/SD存储设备是一种典型的块设备，Linux内核设计了MMC子系统，用于管理MMC/SD设备

MMC驱动子系统包含三个部分：
+ MMC总线(mmc_bus)
+ 封装在platform_device下的host设备
+ 依附在MMC总线的MMC驱动(mmc_driver)

MMC子系统的框架结构如下图所示

core layer根据MMC/SD协议标准实现了协议，card layer与Linux的块设备子系统对接，实现块设备驱动以及完成请求，具体协议经过core layer的接口，最终通过host layer完成传输，对MMC设备进行实际的操作

host和card可以理解为MMC device的MMC主设备和MMC从设备，其中host为集成于芯片内部的MMC controller，card为MMC设备内部实际的存储设备

Linux内核中，使用两个结构体`struct mmc_host`和`struct mmc_card`分别描述host和card，其中host设备被封装成platform_device注册到Linux驱动模型中

#### MMC总线的注册
```c
// drivers/mmc/core/bus.c
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

// mmc总线注册
int mmc_register_bus(void)
{
	return bus_register(&mmc_bus_type);
}

// drivers/mmc/core/core.c
// MMC初始化
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

#### MMC驱动注册
在`drivers/mmc/block.c`中，将`mmc_driver`注册到`mmc_bus`对应的总线系统里
```c
static struct mmc_driver = {
	.drv = {
		.name = "mmcblk",
		.pm = &mmc_blk_pm_ops,
	},
	.probe 		= mmc_blk_probe,
	.remove 	= mmc_blk_remove,
	.shutdown	= mmc_blk_shutdown,
};


```



### SDIO总线简介
SDIO(Secure Digital Input and Output)，即安全数字输入输出接口，他是在SD卡接口的基础上发展而来的，可以兼容之前的SD卡，并可以链接SDIO接口设备，比如蓝牙、WiFi、GPS等

**SDIO卡类型**
+ 全速卡：传输速率超过100Mbps，时钟范围0-25MHz
+ 低速卡：时钟范围0-400KHz

**SDIO卡总线模式**
+ SPI模式
+ 1-bit SD传输模式
+ 4-bit SD传输模式
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230129144326.png)

#### SDIO命令
SDIO总线上的设置和控制都是通过命令来实现的，SDIO总线上都是HOST端发起请求，然后DEVICE端回应请求，其中请求和应答中会包含数据信息：
+ **Command：** 用于开始传输的命令，是由HOST端发往DEVICE端的，其中命令是通过CMD信号线传送的
+ **Response：** DEVICE返回的应答，也是通过CMD信号线传送的
+ **Data：** 数据是双向传送的，可以设置为1线模式，也可以设置为4线模式，数据是通过DAT0-DAT3信号线传输的

**命令格式**
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230129145056.png)
+ Start Bit：起始位，固定为0
+ Transmission：传输方向，值为1表示由host发出，0则表示由device发出
+ Command Index：代表命令索引，例如CMD5的值为5，范围为0-63
+ Argument：CMD所附带的一些参数，不同的CMD，这32bit每一位所代表的含义是不一样的
+ CRC7：7位CRC校验值
+ END：结束位，固定为1

> [官网文档下载地址](https://www.sdcard.org/downloads/pls/)

### MMC子系统介绍
Linux内核中，MMC不仅仅是一个驱动，而是一个子系统，内核把mmc、sd以及sdio三者的驱动代码整合在一起，俗称MMC子系统，源码位于drivers/mmc目录下，mmc目录下有core和host两个文件夹
+ host：针对不同主机端的SDIO、MMC控制器的驱动，这部分需要由驱动工程师来完成
+ core：整个MMC的核心层，这部分实现了不通协议和规范，为HOST层和设备驱动层提供接口函数，还存放了块设备的相关驱动

Linux内核中，使用两个结构体`struct mmc_host`和`struct mmc_card`分别描述host和card，其中host设备被封装成platform_device注册到Linux驱动模型中，整体而言，Linux驱动模型框架下，MMC驱动子系统包括三个部分：
+ MMC总线（mmc_bus）
+ 封装在`platform_device`下的host设备
+ 依附于MMC总线的MMC驱动（mmc_driver）

#### MMC总线注册
MMC总线的注册和platform总线的注册方法相同，均是调用`bus_register()`函数
```c
// drivers/mmc/core/bus.c
// mmc总线定义
static struct bus_type mmc_bus_type = {
    .name       = "mmc",
    .dev_groups = mmc_dev_groups,
    .match      = mmc_bus_match,
    .uevent     = mmc_bus_uevent,
    .probe      = mmc_bus_probe,
    .remove     = mmc_bus_remove,
    .shutdown   = mmc_bus_shutdown,
    .pm         = &mmc_bus_pm_ops,
};

// mmc总线注册
int mmc_register_bus(void)
{
    return bus_register(&mmc_bus_type);
}

// drivers/mmc/core/core.c
static int __init mmc_init(void)
{
    int ret;
    
    ret = mmc_register_bus();
    if (ret)
        return ret;

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

// drivers/mmc/core/host.c
// mmc_host_class注册
static struct class mmc_host_class = {
    .name = "mmc_host",
    .dev_release = mmc_host_classdev_release,
};

int mmc_register_host_class(void)
{
    return class_register(&mmc_host_class);
}
```
主要包括两个方面：
+ 利用`bus_register()`注册mmc_bus，对应sysfs下的`/sys/bus/mmc/`目录
+ 利用`class_register()`注册mmc_host_class，对应`sysfs下的/sys/class/mmc_host`目录

#### MMC驱动注册
`drivers/mmc/core/block.c`中将`mmc_driver`注册到`mmc_bus`对应的总线系统里，主要步骤包括：
+ 通过`register_blkdev()`向内核注册块设备
+ 通过`driver_register()`将`mmc_driver`注册到`mmc_bus`总线系统

`mmc_driver`注册完成之后，会在sysfs中建立目录`/sys/bus/mmc/drivers/mmcblk`
```c
// drivers/mmc/core/block.c
static struct mmc_driver mmc_driver = {
    .drv = {
        .name = "mmcblk",
        .pm = &mmc_blk_pm_ops,
    },
    .probe      = mmc_blk_probe,
    .remove     = mmc_blk_remove,
    .shutdown   = mmc_blk_shutdown,
};

// drivers/mmc/core/bus.h
struct mmc_driver {
    struct device_driver drv;
    int (*probe)(struct mmc_card *card);
    void (*remove)(struct mmc_card *card);
    void (*shutdown)(struct mmc_card *card);
};

// drivers/mmc/core/bus.c
int mmc_register_driver(struct mmc_driver *drv)
{
    drv->drv.bus = &mmc_bus_type;
    return driver_register(&drv->drv);
}

static int __init mmc_blk_init(void)
{
	int res;

	res  = bus_register(&mmc_rpmb_bus_type);
	if (res < 0) {
		pr_err("mmcblk: could not register RPMB bus type\n");
		return res;
	}
	res = alloc_chrdev_region(&mmc_rpmb_devt, 0, MAX_DEVICES, "rpmb");
	if (res < 0) {
		pr_err("mmcblk: failed to allocate rpmb chrdev region\n");
		goto out_bus_unreg;
	}

	if (perdev_minors != CONFIG_MMC_BLOCK_MINORS)
		pr_info("mmcblk: using %d minors per device\n", perdev_minors);

	max_devices = min(MAX_DEVICES, (1 << MINORBITS) / perdev_minors);

	res = register_blkdev(MMC_BLOCK_MAJOR, "mmc");
	if (res)
		goto out_chrdev_unreg;

	res = mmc_register_driver(&mmc_driver);
	if (res)
		goto out_blkdev_unreg;

	return 0;

out_blkdev_unreg:
	unregister_blkdev(MMC_BLOCK_MAJOR, "mmc");
out_chrdev_unreg:
	unregister_chrdev_region(mmc_rpmb_devt, MAX_DEVICES);
out_bus_unreg:
	bus_unregister(&mmc_rpmb_bus_type);
	return res;
}
```

#### MMC设备的注册
MMC设备主要包括主设备host和从设备card两部分，而主设备host将被封装在platform_device中注册到驱动模型中。
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
	int			rescan_entered;	/* used with nonremovable devices */

	int			need_retune;	/* re-tuning is needed */
	int			hold_retune;	/* hold off re-tuning */
	unsigned int		retune_period;	/* re-tuning period in secs */
	struct timer_list	retune_timer;	/* for periodic re-tuning */

	bool			trigger_card_event; /* card_event necessary */

	struct mmc_card		*card;		/* device attached to this host */

	wait_queue_head_t	wq;
	struct mmc_ctx		*claimer;	/* context that has host claimed */
	int			claim_cnt;	/* "claim" nesting count */
	struct mmc_ctx		default_ctx;	/* default context */

	struct delayed_work	detect;
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






