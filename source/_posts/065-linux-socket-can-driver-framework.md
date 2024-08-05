---
title: 065-linux-socket-can-driver-framework
date: 2024-04-26 11:50:36
tags:
---

### phytium E2000 CAN接口特性
E2000Q集成的CAN控制器兼容CAN2.0标准协议和ISO 11898-1(2015) CAN FD标准协议
+ 支持 data frame、remote frame、error frame、overload frame 帧格式
+ CAN 模式下波特率支持 5Kbps 至 1Mbps 配置;
+ CAN FD 模式下仲裁场波特率支持 5Kbps 至 1Mbps 配置,数据场波特率支持 5Kbps至 5Mbps 配置(配置波特率时需满足仲裁场波特率小于或等于数据场波特率);

### phytium socket can驱动
`struct can_priv`结构体
```c
// include/linux/can/dev.h
/*
 * CAN common private data
 */
struct can_priv {
	struct net_device *dev;
	struct can_device_stats can_stats;

	const struct can_bittiming_const *bittiming_const,
		*data_bittiming_const;
	struct can_bittiming bittiming, data_bittiming;
	const struct can_tdc_const *tdc_const;
	struct can_tdc tdc;

	unsigned int bitrate_const_cnt;
	const u32 *bitrate_const;
	const u32 *data_bitrate_const;
	unsigned int data_bitrate_const_cnt;
	u32 bitrate_max;
	struct can_clock clock;

	unsigned int termination_const_cnt;
	const u16 *termination_const;
	u16 termination;
	struct gpio_desc *termination_gpio;
	u16 termination_gpio_ohms[CAN_TERMINATION_GPIO_MAX];

	unsigned int echo_skb_max;
	struct sk_buff **echo_skb;

	enum can_state state;

	/* CAN controller features - see include/uapi/linux/can/netlink.h */
	u32 ctrlmode;		/* current options setting */
	u32 ctrlmode_supported;	/* options that can be modified by netlink */

	int restart_ms;
	struct delayed_work restart_work;

	int (*do_set_bittiming)(struct net_device *dev);
	int (*do_set_data_bittiming)(struct net_device *dev);
	int (*do_set_mode)(struct net_device *dev, enum can_mode mode);
	int (*do_set_termination)(struct net_device *dev, u16 term);
	int (*do_get_state)(const struct net_device *dev,
			    enum can_state *state);
	int (*do_get_berr_counter)(const struct net_device *dev,
				   struct can_berr_counter *bec);
	int (*do_get_auto_tdcv)(const struct net_device *dev, u32 *tdcv);
};
```

`struct phytium_can_plat`结构体
```c
// drivers/net/can/phytium/phytium_can_platform.c
struct phytium_can_plat {
	struct phytium_can_dev cdev;
	struct phytium_can_devtype *devtype;

	int irq;
	void __iomem *reg_base;	
};
```

`struct phytium_can_dev`结构体
```c
// drivers/net/can/phytium/phytium_can.h
struct phytium_can_dev {
	struct can_priv can;

	struct napi_struct napi;
	struct net_device *net;
	struct device *dev;
	struct clk *clk;

	struct sk_buff *tx_skb;
	
	const struct can_bittiming_const *bit_timing;
	spinlock_t lock;
	/* 是否为CANFD模式，设备树没配置的话默认是普通CAN模式 */
	int fdmode;
	u32 isr;
	u32 tx_fifo_depth;
	/* 停止发送标志 */
	unsigned int is_stop_queue_flag;
	void __iomem *base;
	/* CAN发送超时定时器 */
	struct timer_list timer;
	u32 is_tx_done;
	u32 is_need_stop_xmit;
};
```

phytium can设备树节点
```c
can0: can@2800a000 {
	compatible = "phytium,canfd";
	reg = <0x0 0x2800a000 0x0 0x1000>;
	interrupts = <GIC_SPI 81 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&sysclk_200mhz>;
	clock-names = "can_clk";
	tx-fifo-depth = <64>;
	rx-fifo-depth = <64>;
	status = "disabled";
};
```

#### phytium CAN platform驱动probe过程

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240524105540.png)

这里主要是`alloc_candev()`难懂一些，借助这个接口来分配一个`struct net_device`，`net_device`结构体中还包含一个priv的部分，这部分主要由下面三个部分组成
```c
// drivers/net/can/dev/dev.c line 242
/*
 * +-------------------------+
 * | driver's priv           |
 * +-------------------------+
 * | struct can_ml_priv      |
 * +-------------------------+
 * | array of struct sk_buff |
 * +-------------------------+
/*
```
其中`driver's priv`即对应我们驱动定义的结构体`struct phytium_can_plat`，在`struct phytium_can_plat`中`struct can_priv`必须放在第一个位置，在`alloc_candev_mqs()`中还会进一步对`struct can_priv`进行一些初始化

其中涉及到向cansocket层提供的一些接口
```c
/* struct can_priv中涉及到的一些接口 */
cdev->can.do_set_mode = phytium_can_set_mode;
cdev->can.do_get_berr_counter = phytium_can_get_berr_counter;

cdev->can.ctrlmode_supported = CAN_CTRLMODE_LISTENONLY |
					CAN_CTRLMODE_BERR_REPORTING;
cdev->can.bittiming_const = cdev->bit_timing;

if (cdev->fdmode) {
	cdev->can.ctrlmode_supported |= CAN_CTRLMODE_FD;
	dev->mtu = CANFD_MTU;
	cdev->can.ctrlmode = CAN_CTRLMODE_FD;
	cdev->can.data_bittiming_const = cdev->bit_timing;
}

/* net_device的一些操作集函数 */
static const struct net_device_ops phytium_can_netdev_ops = {
	.ndo_open = phytium_can_open,
	.ndo_stop = phytium_can_close,
	.ndo_start_xmit = phytium_can_start_xmit,
	.ndo_change_mtu = can_change_mtu,
};
```

#### CAN数据接收过程
phytium CAN驱动也采取了NAPI的方式进行数据接收，NAPI接收的方式为首次数据包接收依靠中断来触发，接收中断发生后，驱动程序禁止接受中断，通过轮询的方式读取设备的接收缓冲区，读取完毕后再次使能接收中断。NAPI技术适用于高速率短数据包的处理，防止直接使用中断收包导致CPU一直陷入硬中断没时间来处理其他进程。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240524133939.png)

#### CAN数据发送过程
CAN数据发送会调用到`net_device`的ops接口`ndo_start_xmit`，即驱动里面的`phytium_can_start_xmit()`，该接口每次会发送一个CAN帧，并会判断当前发送fifo的大小是否满足下一帧的发送需求，若fifo不够了，则会先停止发送，等收到发送完成中断后再继续判断fifo的大小，重新启动发送。

**发送逻辑的缺陷**：现有代码的CAN数据发送逻辑是嵌入式研发部之前对发送速率不达标提出的修改版本，之前的发送逻辑是发送一帧后先停止发下一帧，等待发送完成中断之后，再继续下一帧发送，这样可以确保CAN帧被准确的发送出去了。现有的发送逻辑，不会对等待上一帧的发送完成中断，马上又开始发送下一帧，直到fifo大小不够再停止发送，这样虽然可以提高发送效率，但是没有判断CAN帧是否被正确发送了，发送逻辑可能还有待优化

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240524142452.png)
