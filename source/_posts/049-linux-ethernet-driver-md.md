---
title: 049_linux_ethernet_driver.md
date: 2023-08-25 13:43:06
tags:
  - Linux
  - Ethernet
categories: Linux
---

### MAC和PHY介绍
以太网接口电路主要由MAC（Media Access Control）控制器和物理层接口PHY（Physical Layer）两大部分组成如下图所示：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230825134627.png)

DMA控制器属于CPU的一部分，DMA控制器可能会参与到网口数据传输中，以上三部分并不一定是独立分开的，PHY整合了大量模拟硬件，MAC是全数字器件，考虑到芯片面积和模数混合架构的原因，通常将MAC集成进微控制器而将PHY留在片外，如下图所示：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230825135021.png)

MAC和PHY工作在OSI七层模型的数据链路层和物理层，具体如下：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230825135105.png)

#### 什么是MAC
MAC即介质访问控制，该部分有两个概念：MAC是一个硬件控制器及MAC通信协议，主要负责控制与物理层连接的物理介质，MAC的硬件大概如下图所示

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230825135220.png)

在发送数据的时候，MAC协议可以事先判断是否可以发送数据，如果可以发送，将数据加上一些控制信息，最终将数据以及控制信息以规定的格式发送到物理层；在接收数据的时候，MAC协议首先判断输入的信息是否发生传输错误，如果没有错误，则去掉控制信息发送到LLC（逻辑链路控制子层）。该层协议是由以太网IEEE-802.3标准定义的。

以太网数据链路层包含MAC（介质访问控制子层）和LLC（逻辑链路控制子层），以太网MAC芯片一端连接CPU，另一端连接到PHY芯片上，它们之间是通过MII接口链接的。

#### 什么是PHY
PHY（Physical Layer）是IEEE802.3中定义的一个标准模块,STA（Station Management Entity）管理实体，一般是MAC通过SMI（Serial Manage Interface）对PHY的行为、状态进行管理和控制，具体的管理和控制动作是通过读写PHY内部的寄存器实现的。PHY是物理接口收发器，它实现了OSI模型的物理层，PHY的基本结构如下图

![](https://raw.githubusercontent.com/JackHuang021/images/master/20230825142814.png)

#### 什么是MII

### 驱动实例分析，基于macb驱动
基于linux 5.10内核的macb驱动

#### 重要数据结构
```c
// drivers/net/ethernet/cadence/macb.h
struct macb {
	void __iomem		*regs;
	bool			native_io;

	/* hardware IO accessors */
	u32	(*macb_reg_readl)(struct macb *bp, int offset);
	void	(*macb_reg_writel)(struct macb *bp, int offset, u32 value);

	size_t			rx_buffer_size;

	unsigned int		rx_ring_size;
	unsigned int		tx_ring_size;

	unsigned int		num_queues;
	unsigned int		queue_mask;
	struct macb_queue	queues[MACB_MAX_QUEUES];

	spinlock_t		lock;
	struct platform_device	*pdev;
	struct clk		*pclk;
	struct clk		*hclk;
	struct clk		*tx_clk;
	struct clk		*rx_clk;
	struct clk		*tsu_clk;
	struct net_device	*dev;
	struct ncsi_dev		*ndev;
	union {
		struct macb_stats	macb;
		struct gem_stats	gem;
	}			hw_stats;

	struct macb_or_gem_ops	macbgem_ops;

	struct mii_bus		*mii_bus;
	struct phylink		*phylink;
	struct phylink_config	phylink_config;
	struct phylink_pcs	phylink_pcs;
	int			link;
	int			speed;
	int			duplex;
	int			use_ncsi;

	int 		force_phy_mode;
	u32			caps;
	unsigned int		dma_burst_length;

	phy_interface_t		phy_interface;

	/* AT91RM9200 transmit queue (1 on wire + 1 queued) */
	struct macb_tx_skb	rm9200_txq[2];
	unsigned int		rm9200_tx_tail;
	unsigned int		rm9200_tx_len;
	unsigned int		max_tx_length;

	u64			ethtool_stats[GEM_STATS_LEN + QUEUE_STATS_LEN * MACB_MAX_QUEUES];

	unsigned int		rx_frm_len_mask;
	unsigned int		jumbo_max_len;

	u32			wol;

	struct macb_ptp_info	*ptp_info;	/* macb-ptp interface */
#ifdef MACB_EXT_DESC
	uint8_t hw_dma_cap;
#endif
	spinlock_t tsu_clk_lock; /* gem tsu clock locking */
	unsigned int tsu_rate;
	struct ptp_clock *ptp_clock;
	struct ptp_clock_info ptp_clock_info;
	struct tsu_incr tsu_incr;
	struct hwtstamp_config tstamp_config;

	/* RX queue filer rule set*/
	struct ethtool_rx_fs_list rx_fs_list;
	spinlock_t rx_fs_lock;
	unsigned int max_tuples;

	struct tasklet_struct	hresp_err_tasklet;

	int	rx_bd_rd_prefetch;
	int	tx_bd_rd_prefetch;

	u32	rx_intr_mask;

	struct macb_pm_data pm_data;

	void (*sel_clk_hw)(struct macb *bp, int speed);

#ifdef CONFIG_MACB_TSN
	u32 tsn_cap;
	struct device_attribute cb_setting_attr;
	struct device_attribute qci_setting_attr;
	struct device_attribute qav_setting_attr;
	struct device_attribute qbv_setting_attr;
	struct device_attribute stream_id_attr;
	u32 num_cb_streams;
	u32 cb_history_len;
#endif
};

// drivers/net/ethernet/cadence/macb.h
struct macb_config {
	u32			caps;
	unsigned int		dma_burst_length;
	int	(*clk_init)(struct platform_device *pdev, struct clk **pclk,
			    struct clk **hclk, struct clk **tx_clk,
			    struct clk **rx_clk, struct clk **tsu_clk);
	int	(*init)(struct platform_device *pdev);
	int	jumbo_max_len;
	void (*sel_clk_hw)(struct macb *bp, int speed);
};
```

#### macb_probe分析
```c
// drivers/net/ethernet/cadence/macb_main.c
static int macb_probe(struct platform_device *pdev)
{
	const struct macb_config *macb_config = &default_gem_config;
	int (*clk_init)(struct platform_device *, struct clk **,
			struct clk **, struct clk **,  struct clk **,
			struct clk **) = macb_config->clk_init;
	int (*init)(struct platform_device *) = macb_config->init;
	struct device_node *np = pdev->dev.of_node;
	struct clk *pclk, *hclk = NULL, *tx_clk = NULL, *rx_clk = NULL;
	struct clk *tsu_clk = NULL;
	unsigned int queue_mask, num_queues;
	bool native_io;
	struct net_device *dev;
	struct resource *regs;
	void __iomem *mem;
	const char *mac;
	struct macb *bp;
	int err, val;

	regs = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	mem = devm_ioremap_resource(&pdev->dev, regs);
	if (IS_ERR(mem))
		return PTR_ERR(mem);

	if (np) {
		const struct of_device_id *match;

		match = of_match_node(macb_dt_ids, np);
		if (match && match->data) {
			macb_config = match->data;
			clk_init = macb_config->clk_init;
			init = macb_config->init;
		}
	} else if (has_acpi_companion(&pdev->dev)) {
		const struct acpi_device_id *match;

		match = acpi_match_device(macb_acpi_ids, &pdev->dev);
		if (match && match->driver_data) {
			macb_config = (void *)match->driver_data;
			clk_init = macb_config->clk_init;
			init = macb_config->init;
		}
	}

	err = clk_init(pdev, &pclk, &hclk, &tx_clk, &rx_clk, &tsu_clk);
	if (err)
		return err;

	pm_runtime_set_autosuspend_delay(&pdev->dev, MACB_PM_TIMEOUT);
	pm_runtime_use_autosuspend(&pdev->dev);
	pm_runtime_get_noresume(&pdev->dev);
	pm_runtime_set_active(&pdev->dev);
	pm_runtime_enable(&pdev->dev);
	native_io = hw_is_native_io(mem);

	macb_probe_queues(mem, native_io, &queue_mask, &num_queues);
	dev = alloc_etherdev_mq(sizeof(*bp), num_queues);
	if (!dev) {
		err = -ENOMEM;
		goto err_disable_clocks;
	}

	dev->base_addr = regs->start;

	SET_NETDEV_DEV(dev, &pdev->dev);

	bp = netdev_priv(dev);
	bp->pdev = pdev;
	bp->dev = dev;
	bp->regs = mem;
	bp->native_io = native_io;
	if (native_io) {
		bp->macb_reg_readl = hw_readl_native;
		bp->macb_reg_writel = hw_writel_native;
	} else {
		bp->macb_reg_readl = hw_readl;
		bp->macb_reg_writel = hw_writel;
	}
	bp->num_queues = num_queues;
	bp->queue_mask = queue_mask;
	if (macb_config)
		bp->dma_burst_length = macb_config->dma_burst_length;
	bp->pclk = pclk;
	bp->hclk = hclk;
	bp->tx_clk = tx_clk;
	bp->rx_clk = rx_clk;
	bp->tsu_clk = tsu_clk;
	if (macb_config)
		bp->jumbo_max_len = macb_config->jumbo_max_len;

	if (macb_config)
		bp->sel_clk_hw = macb_config->sel_clk_hw;

	bp->wol = 0;
	if (device_property_read_bool(&pdev->dev, "magic-packet"))
		bp->wol |= MACB_WOL_HAS_MAGIC_PACKET;
	device_set_wakeup_capable(&pdev->dev, bp->wol & MACB_WOL_HAS_MAGIC_PACKET);

	spin_lock_init(&bp->lock);

	/* setup capabilities */
	macb_configure_caps(bp, macb_config);

#ifdef CONFIG_ARCH_DMA_ADDR_T_64BIT
	if (GEM_BFEXT(DAW64, gem_readl(bp, DCFG6))) {
		dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(44));
		bp->hw_dma_cap |= HW_DMA_CAP_64B;
	}
#endif
	platform_set_drvdata(pdev, dev);

	dev->irq = platform_get_irq(pdev, 0);
	if (dev->irq < 0) {
		err = dev->irq;
		goto err_out_free_netdev;
	}

	/* MTU range: 68 - 1500 or 10240 */
	dev->min_mtu = GEM_MTU_MIN_SIZE;
	if (bp->caps & MACB_CAPS_JUMBO)
		dev->max_mtu = gem_readl(bp, JML) - ETH_HLEN - ETH_FCS_LEN;
	else
		dev->max_mtu = ETH_DATA_LEN;

	if (bp->caps & MACB_CAPS_BD_RD_PREFETCH) {
		val = GEM_BFEXT(RXBD_RDBUFF, gem_readl(bp, DCFG10));
		if (val)
			bp->rx_bd_rd_prefetch = (2 << (val - 1)) *
						macb_dma_desc_get_size(bp);

		val = GEM_BFEXT(TXBD_RDBUFF, gem_readl(bp, DCFG10));
		if (val)
			bp->tx_bd_rd_prefetch = (2 << (val - 1)) *
						macb_dma_desc_get_size(bp);
	}

	bp->rx_intr_mask = MACB_RX_INT_FLAGS;
	if (bp->caps & MACB_CAPS_NEEDS_RSTONUBR)
		bp->rx_intr_mask |= MACB_BIT(RXUBR);

	mac = of_get_mac_address(np);
	if (PTR_ERR(mac) == -EPROBE_DEFER) {
		err = -EPROBE_DEFER;
		goto err_out_free_netdev;
	} else if (!IS_ERR_OR_NULL(mac)) {
		ether_addr_copy(bp->dev->dev_addr, mac);
	} else {
		macb_get_hwaddr(bp);
	}

	err = macb_get_phy_mode(pdev);
	if (err < 0)
		bp->phy_interface = PHY_INTERFACE_MODE_MII;
	else
		bp->phy_interface = err;

	bp->link = 0;
	bp->duplex = DUPLEX_UNKNOWN;
	bp->speed = SPEED_UNKNOWN;

	/* IP specific init */
	err = init(pdev);
	if (err)
		goto err_out_free_netdev;

	if (device_property_read_bool(&pdev->dev, "force-phy-mode")) {
		bp->force_phy_mode = 1;
	}

	err = macb_mii_init(bp);
	if (err)
		goto err_out_free_netdev;

	if (device_property_read_bool(&pdev->dev, "use-ncsi")) {
		if (!IS_ENABLED(CONFIG_NET_NCSI)) {
			dev_err(&pdev->dev, "NCSI stack not enabled\n");
			goto err_out_free_netdev;
		}
		dev_notice(&pdev->dev, "Using NCSI interface\n");
		bp->use_ncsi = 1;
		bp->ndev = ncsi_register_dev(dev, gem_ncsi_handler);
		if (!bp->ndev)
			goto err_out_free_netdev;
	} else {
		bp->use_ncsi = 0;
	}

	netif_carrier_off(dev);

	err = register_netdev(dev);
	if (err) {
		dev_err(&pdev->dev, "Cannot register net device, aborting.\n");
		goto err_out_unregister_mdio;
	}

	tasklet_setup(&bp->hresp_err_tasklet, macb_hresp_error_task);

	netdev_info(dev, "Cadence %s rev 0x%08x at 0x%08lx irq %d (%pM)\n",
		    macb_is_gem(bp) ? "GEM" : "MACB", macb_readl(bp, MID),
		    dev->base_addr, dev->irq, dev->dev_addr);

	pm_runtime_mark_last_busy(&bp->pdev->dev);
	pm_runtime_put_autosuspend(&bp->pdev->dev);

	return 0;

err_out_unregister_mdio:
	mdiobus_unregister(bp->mii_bus);
	mdiobus_free(bp->mii_bus);

err_out_free_netdev:
	free_netdev(dev);

err_disable_clocks:
	clk_disable_unprepare(tx_clk);
	clk_disable_unprepare(hclk);
	clk_disable_unprepare(pclk);
	clk_disable_unprepare(rx_clk);
	clk_disable_unprepare(tsu_clk);
	pm_runtime_disable(&pdev->dev);
	pm_runtime_set_suspended(&pdev->dev);
	pm_runtime_dont_use_autosuspend(&pdev->dev);

	return err;
}
```

