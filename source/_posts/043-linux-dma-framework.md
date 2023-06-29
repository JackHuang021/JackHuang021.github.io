---
title: Linux DMA驱动框架
date: 2023-05-09 16:14:27
tags:
  - Linux
  - DMA
categories: Linux
---

#### 相关概念
DMA是Direct Memory Access的缩写，就是绕开CPU进行内存的访问，DMA控制器就是用来协助CPU在memory和memory或者memory和设备之间搬运数据

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230526141809.png)
##### DMA channels
一个DMA可以“同时”进行DMA传输的个数是有限的，这称作DMA channels，这里的channel只是一个逻辑上的概念
> 鉴于总线访问的冲突，以及内存一致性的考量，从物理的角度看，不大可能会同时进行两个（及以上）的DMA传输。因而DMA channel不太可能是物理上独立的通道；
> 
> 很多时候，DMA channels是DMA controller为了方便，抽象出来的概念，让consumer以为独占了一个channel，实际上所有channel的DMA传输请求都会在DMA controller中进行仲裁，进而串行传输；
> 
> 因此，软件也可以基于controller提供的channel（我们称为“物理”channel），自行抽象更多的“逻辑”channel，软件会管理这些逻辑channel上的传输请求。实际上很多平台都这样做了，在DMA Engine framework中，不会区分这两种channel（本质上没区别）。

##### DMA request line
DMA传输是由CPU发起的，CPU会告诉DMA控制器，把xxx地方的数据搬运到xxx地方，而DMA控制器，除了负责怎么搬之外还要决定一件非常重要的事情：何时开始搬运？

因为，CPU发起DMA传输的时候，并不知道当前是否具备传输条件，例如source设备是否有数据、dest设备的FIFO是否空闲等等。那谁知道是否可以传输呢？设备！因此，需要DMA传输的设备和DMA控制器之间，会有几条物理的连接线（称作DMA request，DRQ），用于通知DMA控制器可以开始传输了。

通常来说，每一个数据收发的节点（称作endpoint），和DMA controller之间，就有一条DMA request line。


##### 传输参数
**transfer size:** 在每一个时钟周期，DMA controller将1 byte的数据从一个buffer搬到另一个buffer，直到搬完transfer size个byte即可停止

**transfer width:** 传输的数据宽度，在一个时钟周期中，传输指定的bit的数据，DDMA固定为4字节

**buffer size:** DMA控制器内部可缓存的数据量大小

**scatter-gather:** DMA传输一般情况下只能处理物理上连续的buffer，在某些场景下将一些非连续的buffer拷贝到一个连续的buffer中，这样的操作称为scatter-gather，对于这种非连续的传输，大多时候都是通过软件，将传输分成多个连续的小块（chunk）,例如在dmaengine中的scatterlist

**burst size:** DMA控制器内部可缓存的数据量大小，按照DDMA的手册描述应该是固定为64字节，一次搬64字节

#### Linux dmaengine
从方向上来说：DMA传输可以分为4类，memory到memory，memory到device，device到memory以及device到device，从linux kernel的角度，外设都是slave，因此这些有device参与的传输（MEM2DEV, DEV2MEM, DEV2DEV）为Slave-DMA传输，另一种memory到memory的传输，被称为Async TX

因为Linux为了方便基于DMA的memcpy、memset等操作，在dma engine之上，封装了一层更为简洁的API，这种API就是Async TX API（以async_开头，例如async_memcpy、async_memset、async_xor等）。因为memory到memory的DMA传输有了比较简洁的API，没必要直接使用dma engine提供的API，最后就导致dma engine所提供的API就特指为Slave-DMA API

Slave-DMA中的slave指的是参与DMA传输的设备，对应的master就是指DMA controller自身

##### 重要数据结构
`struct dma_device`，用于抽象dma controller
```c
/* include/linux/dmaengine.h */
struct dma_device {
	struct kref ref;
	/* 当前支持的dma通道数，读取设备树的dma-channels值 */
	unsigned int chancnt;
	/* 已经申请的dma通道数 */
	unsigned int privatecnt;
  	/* 用于保存该controller支持的所有dma channel */
  	/* 初始化的时候需要将所有的channel加入到链表头中 */
	struct list_head channels;
	struct list_head global_node;
	struct dma_filter filter;
  	/* 用于指示dma controller所具备的功能，DDMA仅具有DMA_SLAVE的功能 */
	dma_cap_mask_t  cap_mask;

	/* dma controller支持的源数据宽度类型，DDMA只支持4字节 */
	u32 src_addr_widths;
	/* dma controller支持的目的数据宽度类型，DDMA只支持4字节 */
	u32 dst_addr_widths;
	/* dma controller支持的传输方向，DDMA只支持MEM_TO_DEV DEV_TO_MEM */
	u32 directions;
	u32 min_burst;
	u32 max_burst;
	/* 单次传输的一个散列表中支持的最大长度，DDMA驱动里面设置是4KB */
	u32 max_sg_burst;
	/* 报告传输剩余数据长度的功能 */
	bool descriptor_reuse;
	/* 剩余数据长度报告的类型，一共有三种：无法上报、传输完成后即可上报、每个brust传输后报告 */
	enum dma_residue_granularity residue_granularity;

	/* 分配通道的接口 */
	int (*device_alloc_chan_resources)(struct dma_chan *chan);
	/* 释放通道的接口 */
	void (*device_free_chan_resources)(struct dma_chan *chan);

	/* dma通道准备传输的接口 */
	struct dma_async_tx_descriptor *(*device_prep_slave_sg)(
		struct dma_chan *chan, struct scatterlist *sgl,
		unsigned int sg_len, enum dma_transfer_direction direction,
		unsigned long flags, void *context);

	void (*device_caps)(struct dma_chan *chan,
			    struct dma_slave_caps *caps);
	/* 对dma通道进行配置的接口 */
	int (*device_config)(struct dma_chan *chan,
			     struct dma_slave_config *config);
	/* 暂停传输的接口 */
	int (*device_pause)(struct dma_chan *chan);
	/* 恢复传输的接口 */
	int (*device_resume)(struct dma_chan *chan);
	/* 终止dma通道传输的接口 */
	int (*device_terminate_all)(struct dma_chan *chan);
	/* 获取本次传输状态和已经传输完的长度 */
	enum dma_status (*device_tx_status)(struct dma_chan *chan,
					    dma_cookie_t cookie,
					    struct dma_tx_state *txstate);
	/* 启动dma传输的接口 */
	void (*device_issue_pending)(struct dma_chan *chan);
	void (*device_release)(struct dma_device *dev);
	/* debugfs support */
#ifdef CONFIG_DEBUG_FS
	void (*dbg_summary_show)(struct seq_file *s, struct dma_device *dev);
	struct dentry *dbg_dev_root;
#endif
};
```

`struct dma_chan`用于抽象物理dma channel
```c
/* include/linux/dmaengine.h */
struct dma_chan {
	/* 指向所在的dma device */
	struct dma_device *device;
	struct device *slave;
	dma_cookie_t cookie;
	/* 在这个channel上最后一次完成的传输的cookie */
	/* dma controller driver可以调用dma_cookie_complete设置它的value */
	dma_cookie_t completed_cookie;

	/* sysfs */
	int chan_id;
	struct dma_chan_dev *dev;
	const char *name;
#ifdef CONFIG_DEBUG_FS
	char *dbg_client_name;
#endif

	/* 用于将该channel添加到dma_device的channels列表中 */
	struct list_head device_node;
	struct dma_chan_percpu __percpu *local;
	int client_count;
	int table_count;

	/* DMA router */
	struct dma_router *router;
	void *route_data;

	void *private;
};
```

`struct virt_dma_chan`用于抽象一个虚拟的dma_channel，多个虚拟channel可以共用一个物理channel，并由软件调度多个传输请求，将多个虚拟channel的传输串行地在物理channel上完成
```c
/* drivers/dma/virt_dma.h */

/* struct virt_dma_desc 对请求描述符做了一个简单的封装 */
struct virt_dma_desc {
	struct dma_async_tx_descriptor tx;
	struct dmaengine_result tx_result;
	/* protected by vc.lock */
	struct list_head node;
};

struct virt_dma_chan {
	/* 指向一个物理通道 */
	struct dma_chan	chan;
	/* 用于等待该虚拟channel上传输的完成，完成后会调度该task */
	struct tasklet_struct task;
	/* 用于传输完成或终止传输后释放描述符内存 */
	void (*desc_free)(struct virt_dma_desc *);

	spinlock_t lock;

	/* protected by vc.lock */
	/* 链表用于保存不同状态下的虚拟channel描述符 */
	struct list_head desc_allocated;
	struct list_head desc_submitted;
	struct list_head desc_issued;
	struct list_head desc_completed;
	struct list_head desc_terminated;

	struct virt_dma_desc *cyclic;
};
```

`struct dma_slave_config`，DMA client对DMA channel的配置结构体
```c
struct dma_slave_config {
	/* 传输方向 DMA_MEM_TO_DEV DMA_DEV_TO_MEM */
	enum dma_transfer_direction direction;
	/* 设备物理地址，dev_to_mem的时候才起作用，通常是固定的外设fifo地址 */
	phys_addr_t src_addr;
	/* 设备物理地址，mem_to_dev的时候才起作用，通常是固定的外设fifo地址 */
	phys_addr_t dst_addr;
	/* 读取DMA数据源寄存器的的地址宽度 */
	enum dma_slave_buswidth src_addr_width;
	/* 读取DMA目标地址寄存器的的地址宽度 */
	enum dma_slave_buswidth dst_addr_width;
	u32 src_maxburst;
	u32 dst_maxburst;
	u32 src_port_window_size;
	u32 dst_port_window_size;
	bool device_fc;
	unsigned int slave_id;
};
```

`struct dma_async_tx_descriptor`，用于描述一次DMA传输，类似一个文件句柄，controller driver返回给client driver一个描述符
```c
struct dma_async_tx_descriptor {
	/* 用于追踪本次传输 */
	dma_cookie_t cookie;
	enum dma_ctrl_flags flags; /* not a 'long' to pack with cookie */
	dma_addr_t phys;
	/* 对应的dma channel */
	struct dma_chan *chan;
	/* controller driver提供的回调函数，用于把该描述符提交到传输列表，由dmaengine调用 */
	dma_cookie_t (*tx_submit)(struct dma_async_tx_descriptor *tx);
	/* 用于释放该描述符的回调函数，dmaengine调用 */
	int (*desc_free)(struct dma_async_tx_descriptor *tx);
	/* 传输完成的callback函数，client driver提供 */
	dma_async_tx_callback callback;
	dma_async_tx_callback_result callback_result;
	void *callback_param;
	struct dmaengine_unmap_data *unmap;
	enum dma_desc_metadata_mode desc_metadata_mode;
	struct dma_descriptor_metadata_ops *metadata_ops;
#ifdef CONFIG_ASYNC_TX_ENABLE_CHANNEL_SWITCH
	struct dma_async_tx_descriptor *next;
	struct dma_async_tx_descriptor *parent;
	spinlock_t lock;
#endif
};
```

#### DMA API使用
##### 从CPU角度看到的地址和从DMA控制器看到的地址
在DMA API中涉及到好几个地址的概念（物理地址、虚拟地址、总线地址）

内核通常使用的地址是虚拟地址，我们调用`kmalloc()`、`vmalloc()`或者类似的接口返回的地址都是虚拟地址

虚拟内存系统（TLB 页表等）将虚拟地址翻译成物理地址，物理地址保存在`phys_addr_t`或`resource_size_t`的变量中，对于一个硬件设备上的寄存器等设备资源，内核是按照物理地址来管理的，通过`/proc/iomem`可以看到这些和设备IO相关的物理地址，驱动不能直接使用这些物理地址，必须首先通过`ioremap()`接口将这些物理地址映射到内核虚拟地址空间上去

I/O设备使用第三种地址：总线地址。如果设备在MMIO地址空间中有若干的寄存器，或者该设备可以通过DMA执行读写系统内存的操作，这种情况下，设备使用的地址就是总线地址

各种地址概念关系图
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230512103247.png)

DMA使用的内存地址：在驱动中可以通过`kmalloc`或者其他类似接口分配一个DMA buffer，并且返回了虚拟地址X，MMU将X地址映射成了物理地址Y，从而定位了DMA buffer在系统内存中的位置，因此驱动可以通过地址X来操作DMA buffer，但是设备并不能通过X地址来访问DMA buffer，因为MMU对设备不可见

驱动在调用`dma_map_single()`这样的接口函数的时候会传递一个虚拟地址X，在这个函数中会设定IOMMU的页表，将地址X映射到Z，并且将返回Z这个总线地址，驱动可以把Z这个总线地址设定到设备上的DMA相关的寄存器中，这样当设备发起对地址Z开始的DMA操作的时候，IOMMU可以进行地址映射，并将DMA操作定位到Y地址开始的DMA buffer

##### DMA内存映射
一致性映射（coherent DMA mappings）是使用专门的接口分配一块DMA缓冲区，这块DMA缓冲区是关闭了cache机制的。也就是数据直接写入内存，这样就不存在一致性问题。

一致性dma映射接口：
```c
void *dma_alloc_coherent(struct device *dev, size_t size,dma_addr_t *dma_handle, gfp_t flag)

dma_addr_t dma_handle;
cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, gfp);
```
dma_alloc_coherent函数返回两个值，一个是从CPU角度访问DMA buffer的虚拟地址，另外一个是从设备（DMA controller）角度看到的bus address：dma_handle，驱动可以将这个bus address传递给DMA控制器。

##### 设备驱动使用dmaengine
对于设备驱动，要基于dmaengine提供的Slave-DMA API进行DMA传输的话，需要如下的操作步骤
1. 申请一个DMA channel
2. 根据设备的特性，配置dma channel的参数
3. 要进行DMA传输的时候，获取一个用于识别本次传输的描述符（descriptor）
4. 将本次传输提交给dma engine并启动传输
5. 等待传输结束

##### 传输描述符
DMA属于异步传输，在启动传输之前，slave driver需要将此次传输的一些信息（src dst的地址，传输的方向）提交给dmaengine，dma controller驱动确认后会返回一个描述符（由`struct dma_async_tx_escriptor`抽象），slave driver就以该描述符为单位，控制并跟踪此次传输

```c
/**
 * @chan: 本次传输所用到的channel
 * @sgl: scatter gather buffers 数组地址
 * @sg_len: scatter gather buffers 数组的长度
 * @direction: 数据传输的方向
 * @flags: 一些额外的标志位
 */
struct dma_async_tx_descriptor *(*device_prep_slave_sg)(struct dma_chan *chan,
			struct scatterlist *sgl, unsigned int sg_len,
			enum dma_transfer_direction direction,
			unsigned long flags, void *context);
```

#### Phytium DDMA驱动
##### 设备树描述
+ dma controller设备树描述
```c
ddma0: ddma@28003000 {
	compatible = "phytium,ddma";
	reg = <0x0 0x28003000 0x0 0x1000>;
	interrupts = <GIC_SPI 75 IRQ_TYPE_LEVEL_HIGH>;
	#dma-cells = <2>;			// 用来表示dma client设备树dmas中属性的个数
	dma-channels = <8>;			// dma 通道个数
	clocks = <&sysclk_50mhz>;
	clock-names = "core_clk";
};

ddma1: ddma@28004000 {
	compatible = "phytium,ddma";
	reg = <0x0 0x28004000 0x0 0x1000>;
	interrupts = <GIC_SPI 76 IRQ_TYPE_LEVEL_HIGH>;
	#dma-cells = <2>;
	dma-channels = <8>;
	clocks = <&sysclk_50mhz>;
	clock-names = "core_clk";
};
```

+ dma client设备树描述
```bash
spi2: spi@2803c000 {
	compatible = "phytium,spi";
	reg = <0x0 0x2803c000 0x0 0x1000>;
	interrupts = <GIC_SPI 161 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&sysclk_48mhz>;
	num-cs = <4>;
	#address-cells = <1>;
	#size-cells = <0>;
	dmas = <&ddma0 0 8>,
			<&ddma0 1 21>;
	dma-names = "tx", "rx";
	status = "disabled";
};
```

##### ddma驱动相关结构体
`struct phytium_ddma_device`用于描述ddma控制器
```c
/**
 * struct phytium_ddma_device - the struct holding info describing DDMA device
 * @dma_dev: an instance for struct dma_device
 * @irq: the irq that DDMA using
 * @base: the mapped register I/O base of this DDMA
 * @core_clk: DDMA clock
 * @dma_channels: the number of DDMA physical channels
 * @chan: the phyical channels of DDMA
 */
struct phytium_ddma_device {
	struct dma_device dma_dev;
	struct device *dev;
	int	irq;
	void __iomem *base;
	struct clk *core_clk;
	u32 dma_channels;
	struct phytium_ddma_chan *chan;
};
```

`phytium_ddma_chan`用于描述一个ddma 物理通道
```c
/**
 * struct phytium_ddma_chan - the struct holding info describing dma channel
 * @vchan: virtual dma channel
 * @base: the mapped register I/O of dma physical channel
 * @id: the id of ddma physical channel
 * @desc: the transform request descriptor
 * @dma_config: config parameters for dma channel
 * @busy: the channel busy flag, this flag set when channel is tansferring
 * @is_used: the channel bind flag, this flag set when channel binded
 * @next_sg: the index of next scatter-gatter
 * @paddr: used to align data between dma provider and consumer
 */
struct phytium_ddma_chan {
	struct virt_dma_chan vchan;
	void __iomem *base;
	u32 id;
	struct phytium_ddma_desc *desc;
	struct dma_slave_config dma_config;
	bool busy;
	bool is_used;
	u32 next_sg;
	dma_addr_t paddr;
	char *buf;
};
```

`struct phytium_ddma_desc`dma传输时使用的描述符，里面记录了本次传输的数据
```c
/**
 * struct phytium_ddma_desc - the struct holding info describing ddma request
 * descriptor
 * @vdesc: ddma request descriptor
 * @num_sgs: the size of scatter-gather list
 * @sg_req: use to save scatter-gather list info
 */
struct phytium_ddma_desc {
	struct virt_dma_desc vdesc;
	u32 num_sgs;
	struct phytium_ddma_sg_req sg_req[];
};
```

`struct phytium_ddma_sg_req`用于记录当前传输的scatter-gather信息，源数据地址、设备数据寄存器地址、传输长度等
```c
/**
 * struct phytium_ddma_sg_req - scatter-gatter list data info
 * @len: number of bytes to transform
 * @mem_addr_l: bus address low 32bit
 * @mem_addr_h: bus address high 32bit
 * @dev_addr: dma cousumer data reg addr
 */
struct phytium_ddma_sg_req {
	u32 len;
	u32 mem_addr_l;
	u32 mem_addr_h;
	u32 dev_addr;
};
```

#### pl011 DMA device驱动
##### pl011 uart rx逻辑
pl011 串口RX逻辑主要借助pl011的串口接收超时中断来进行数据接收，如pl011的手册描述，当rx fifo中不为空且连续32个bit的时间内没收到任何数据则产生接收超时中断，在超时中断中先对dma rx通道进行暂停，并检查dma实际传输了多少长度的数据，将实际传输的数据存入tty缓冲区，再把fifo中剩下的数据读出来，也存入tty缓冲区，这样完成了一次接收，具体代码位于`pl011_dma_rx_irq()`
![](https://raw.githubusercontent.com/JackHuang021/images/master/20230615143556.png)

##### pl011 uart tx逻辑
当串口发送中断触发时（发送fifo中的数据低于设定的触发值）就会进入发送逻辑，dma模式下uart有数据需要发送时会调用`pl011_dma_tx_irq()`走dma的一套数据发送流程进行发送，位于`pl011_dma_tx_refill()`中

#### SPI功能测试
使用spidev-test进行spi硬件回环测试
```bash
root@Ubuntu:~# spidev_test -D /dev/spidev0.0 -v
spi mode: 0x0
bits per word: 8
max speed: 500000 Hz (500 kHz)
TX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF F0 0D  |......@.........................|
RX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF F0 0D  |......@.........................|
```

##### spidev_test测试
测试命令：`spidev_test -s <speed> -D /dev/spidev0.0 -S <length> -I <iterations>`
+ E2000D demo板测试数据
	| 单次传输字节长度 | DMA传输速度 | 中断传输速度 |
	| :-: | :-: | :-: |
	| 64KB | 15.8Mbps | 10.6Mbps |
	| 1KB | 15.2Mbps | 12Mbps |
	| 512B | 14.4Mbps | 11.8Mbps |
	| 128B | 11.1Mbps | 10.8Mbps |
	| 8B | 3.1Mbps | 3.5Mbps |

+ 树莓派4测试数据
	| 测试用例 | DMA传输速度 | 中断传输速度 |
	| :-: | :-: | :-: |
	| 25MHz 单次64KB | 24.6Mbps | 22.6Mbps |
	| 250MHz 单次64KB | 90.2Mbps | 29.1Mbps |

##### 外接flash mtd_speedtest速度测试
+ spi中断传输模式
```bash
root@Ubuntu:~# echo 7 > /proc/sys/kernel/printk
root@Ubuntu:~# modprobe mtd_speedtest dev=0 count=100
[ 1393.267903] 
[ 1393.269405] =================================================
[ 1393.275203] mtd_speedtest: MTD device: 0    count: 100
[ 1393.280379] mtd_speedtest: not NAND flash, assume page size is 512 bytes.
[ 1393.287182] mtd_speedtest: MTD device size 16777216, eraseblock size 4096, page size 512, count of er0
[ 1406.236226] mtd_speedtest: testing eraseblock write speed
[ 1407.918432] mtd_speedtest: eraseblock write speed is 238 KiB/s
[ 1407.924276] mtd_speedtest: testing eraseblock read speed
[ 1408.224498] mtd_speedtest: eraseblock read speed is 1360 KiB/s
[ 1420.949754] mtd_speedtest: testing page write speed
[ 1422.628841] mtd_speedtest: page write speed is 238 KiB/s
[ 1422.634163] mtd_speedtest: testing page read speed
[ 1422.955827] mtd_speedtest: page read speed is 1265 KiB/s
[ 1435.536738] mtd_speedtest: testing 2 page write speed
[ 1437.265931] mtd_speedtest: 2 page write speed is 232 KiB/s
[ 1437.271429] mtd_speedtest: testing 2 page read speed
[ 1437.585886] mtd_speedtest: 2 page read speed is 1294 KiB/s
[ 1437.591385] mtd_speedtest: Testing erase speed
[ 1450.136398] mtd_speedtest: erase speed is 31 KiB/s
[ 1450.141203] mtd_speedtest: Testing 2x multi-block erase speed
[ 1462.805642] mtd_speedtest: 2x multi-block erase speed is 31 KiB/s
[ 1462.811747] mtd_speedtest: Testing 4x multi-block erase speed
[ 1475.535328] mtd_speedtest: 4x multi-block erase speed is 31 KiB/s
[ 1475.541432] mtd_speedtest: Testing 8x multi-block erase speed
[ 1488.202989] mtd_speedtest: 8x multi-block erase speed is 31 KiB/s
[ 1488.209098] mtd_speedtest: Testing 16x multi-block erase speed
[ 1500.921395] mtd_speedtest: 16x multi-block erase speed is 31 KiB/s
[ 1500.927592] mtd_speedtest: Testing 32x multi-block erase speed
[ 1513.647523] mtd_speedtest: 32x multi-block erase speed is 31 KiB/s
[ 1513.653714] mtd_speedtest: Testing 64x multi-block erase speed
[ 1526.322498] mtd_speedtest: 64x multi-block erase speed is 31 KiB/s
[ 1526.328691] mtd_speedtest: finished
[ 1526.332216] =================================================
```

##### 外接flash dd读写测试
+ spi中断传输模式
```
# 写入测试
root@Ubuntu:~# time dd if=/dev/zero of=/dev/mtd0 bs=1024k count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 43.2331 s, 243 kB/s

real    0m43.247s
user    0m0.001s
sys     0m22.231s

# 读测试
root@Ubuntu:~# time dd if=/dev/mtd0 of=/dev/null bs=1024k count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 8.76685 s, 1.2 MB/s

real    0m8.773s
user    0m0.004s
sys     0m0.011s
```

#### 串口功能测试
测试工具： 
+ [tinyserial v1.4](https://github.com/carloscn/tinyserial.git)
+ [linux-serial-test](https://github.com/cbrake/linux-serial-test.git)

4Mbps波特率回环测试，tx和rx数据能对齐
![](https://raw.githubusercontent.com/JackHuang021/images/master/%E4%B8%B2%E5%8F%A3DMA%204Mbps%E4%BC%A0%E8%BE%93.png)

4Mbps发送单个字节（0x55）波形
![](https://raw.githubusercontent.com/JackHuang021/images/master/002.BMP)

#### 引用
> http://www.wowotech.net/tag/dma
> https://www.jianshu.com/p/e1b622234d13