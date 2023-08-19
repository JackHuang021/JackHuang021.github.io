---
title: phytium i2c适配器驱动
author: Jack
tags:
  - Linux
  - I2C
categories:
  - Linux
notshow: true
abbrlink: 1dd4e6b7
---


#### I2C总线驱动
I2C总线驱动重点是I2C适配器（也就是SOC的I2C接口）控制器驱动，这里涉及到两个重要的数据结构：i2c_adapter和i2c_algorithm，Linux内核将SOC的I2C控制器抽象成i2c_adapter，i2c_adapter定义在include/linux/i2c.h中，结构体的内容如下：
<!-- more -->
    ```
    struct i2c_adapter {
        struct module *owner;
        unsigned int class;		  /* classes to allow probing for */
        const struct i2c_algorithm *algo; /* the algorithm to access the bus */
        void *algo_data;

        /* data fields that are valid for all devices	*/
        const struct i2c_lock_operations *lock_ops;
        struct rt_mutex bus_lock;
        struct rt_mutex mux_lock;

        int timeout;			/* in jiffies */
        int retries;
        struct device dev;		/* the adapter device */

        int nr;
        char name[48];
        struct completion dev_released;

        struct mutex userspace_clients_lock;
        struct list_head userspace_clients;

        struct i2c_bus_recovery_info *bus_recovery_info;
        const struct i2c_adapter_quirks *quirks;

        struct irq_domain *host_notify_domain;
    };
    ```
    其中algo成员为i2c适配器对外提供的API读写操作函数，i2c_algorithm为I2C适配器和I2C设备通信的方法，i2c_algorithm结构体的内容如下：
    ```
    struct i2c_algorithm {
        /* If an adapter algorithm can't do I2C-level access, set master_xfer
           to NULL. If an adapter algorithm can do SMBus access, set
           smbus_xfer. If set to NULL, the SMBus protocol is simulated
           using common I2C messages */
        /* master_xfer should return the number of messages successfully
           processed, or a negative value on error */
        int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
                   int num);
        int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr,
                   unsigned short flags, char read_write,
                   u8 command, int size, union i2c_smbus_data *data);

        /* To determine what the adapter supports */
        u32 (*functionality) (struct i2c_adapter *);

    #if IS_ENABLED(CONFIG_I2C_SLAVE)
        int (*reg_slave)(struct i2c_client *client);
        int (*unreg_slave)(struct i2c_client *client);
    #endif
    };
    ```
    + master_xfer是I2C适配器的的传输函数
    + smbus_xfer是SMBUS总线的传输函数

+ I2C适配器驱动的主要工作就是初始化i2c_adapter的结构体变量，然后实现i2c_algorithm中的传输函数，完成后通过*i2c_add_numbered_adapter*或者*i2c_add_adapter*这两个函数向系统注册设置好的i2c_adapter，函数原型如下：
`int i2c_add_adapter(struct i2c_adapter *adapter)`
`int i2c_add_numbered_adapter(struct i2c_adapter *adap)`
i2c_add_adapter()使用动态的总线号，i2c_add_numbered_adapter()使用静态的总线号

#### phytium i2c适配器驱动分析
+ phytium i2c设备树节点内容
    ```
    mio14: i2c@28030000 {
        compatible = "phytium,i2c";
        reg = <0x0 0x28030000 0x0 0x1000>;
        interrupts = <GIC_SPI 106 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&sysclk_50mhz>;
        #address-cells = <1>;
        #size-cells = <0>;
        status = "okay";
    };
    ```
    对应的驱动文件为*drivers/i2c/busses/i2c-phytium-platform.c*

+ phytium i2c适配器驱动为一个标准的platform驱动
    ```
    static struct platform_driver phytium_i2c_driver = {
        .probe = phytium_i2c_plat_probe,
        .remove = phytium_i2c_plat_remove,
        .driver = {
            .name = DRV_NAME,
            .of_match_table = of_match_ptr(phytium_i2c_of_match),
            .acpi_match_table = ACPI_PTR(phytium_i2c_acpi_match),
            .pm = &phytium_i2c_dev_pm_ops,
        },
    };
    ```

+ 使用*of_device_id*与设备树mio14节点相匹配，在platform_match函数中进行匹配
    ```
    static const struct of_device_id phytium_i2c_of_match[] = {
        { .compatible = "phytium,i2c", },
        {   },
    };
    MODULE_DEVICE_TABLE(of, phytium_i2c_of_match);
    ```

+ 当设备和驱动匹配完成后，*phytium_i2c_plat_probe*函数就会执行，完成i2c适配器的初始化工作，probe中的工作如下：
    + 调用platform_get_irq()函数获取中断号
    + 调用platform_get_resource()函数获取I2C控制器的寄存器物理基地址，获取到物理基地址后再使用devm_ioremap_resource()函数对其进行内存映射，得到可以在Linux内核中使用的虚拟内存地址
    + 设置I2C设备总线速度
    + 根据I2C地址第30位数据来判断当前I2C适配器配置成slave模式还是master模式，填充phytium_i2c_dev数据成员，主要设计适配器工作能力等一些参数标志
    + 使能I2C总线时钟
    + 调用i2c_phytium_probe()函数，该函数位于drivers/i2c/busses/i2c-phytium-master.c，在i2c_phytium_probe()里面继续完善adapter数据成员

+ phytium_i2c_dev结构体
    ```
    struct phytium_i2c_dev {
        struct device		*dev;
        void __iomem		*base;
        int			irq;
        u32			flags;
        struct completion	cmd_complete;
        struct clk		*clk;
        struct reset_control	*rst;
        int			mode;
        struct i2c_client	*slave;
        u32			(*get_clk_rate_khz)(struct phytium_i2c_dev *dev);

        struct i2c_adapter	adapter;
        struct i2c_client	*ara;
        struct i2c_smbus_alert_setup alert_data;

        struct phytium_pci_i2c *controller;

        unsigned int		status;
        int			cmd_err;
        u32			abort_source;

        struct i2c_msg		*msgs;
        int			msgs_num;
        int			msg_write_idx;
        int			msg_read_idx;
        int			msg_err;
        u32			tx_buf_len;
        u8			*tx_buf;
        u32			rx_buf_len;
        u8			*rx_buf;

        u32			master_cfg;
        u32			slave_cfg;
        u32			functionality;
        unsigned int		tx_fifo_depth;
        unsigned int		rx_fifo_depth;
        int			rx_outstanding;

        struct i2c_timings	timings;
        u32			sda_hold_time;
        u16			ss_hcnt;
        u16			ss_lcnt;
        u16			fs_hcnt;
        u16			fs_lcnt;
        u16			fp_hcnt;
        u16			fp_lcnt;
        u16			hs_hcnt;
        u16			hs_lcnt;

        bool			pm_disabled;
        void			(*disable)(struct phytium_i2c_dev *dev);
        void			(*disable_int)(struct phytium_i2c_dev *dev);
        int			(*init)(struct phytium_i2c_dev *dev);
    };
    ```


