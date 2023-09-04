---
title: Linux电源管理框架
tags:
  - Linux
  - Power Management Framework
categories: Linux
abbrlink: '43564739'
date: 2023-01-13 08:12:46
---

### 概述
Linux内核中电源管理主要涉及到供电、充电、时钟、频率、电压、睡眠唤醒等内容

<!-- more -->

Linux内核电源管理的组成：
1. Gerneric PM：传统意义上的电源管理，主要是Power off, sleep, hibernate, restart等，涉及到各个层级的suspend, resume, prepare_suspend, suspend_early, suspend_noirq, resume_late, complete
2. Runtime PM和wakelock，kernel层级的调度
3. CPU Idle
4. Clock Framework，时钟资源
5. CPU Freq, Device Freq，CPU和设备频率相关
6. OPP，芯片和设备正常工作的电压和频率组合
7. PM QOS，系统运行状态的质量
8. Regulator Framework，电压和电流
9. Power Supply，电源供电状态

<!-- more -->

### Linux内核电源状态
在Linux内核中，将电源划分为如下几个状态
| ACPI | State | Linux State Drscription |
| :-: | :-: | :- |
| S0 | On | Working |
| S1 | Standby | CPU and RAM are powered but not executed |
| S2 | N/A | N/A |
| S3 | Suspend to RAM | CPU is off, RAM is powered and the running content is saved to RAM |
| S4 | Suspend to Disk | All content is saved to Disk and power down |
| S5 | Shutdown | Shutdown the system |

### power supply子系统
#### power supply软件架构
Linux内核抽象出来power supply子系统为驱动提供了统一的框架，功能包括：
1. 抽象power supply设备的共性，向用户空间提供统一的API
2. 为底层power supply驱动的编写，提供简单统一的方式，同时封装并实现公共逻辑

power supply class位于`driver/power/supply`目录中，主要由3部分组成：
1. power supply core：用于抽象核心数据结构，实现公共逻辑
2. power supply sysfs：实现sysfs以及uevent功能
3. power supply leds：给予Linux led class，提供power supply设备状态指示的通用实现

驱动工程师可以基于power supply class实现具体的power supply driver，主要处理平台相关、硬件相关的逻辑

相关数据结构
```c

/* Description of power supply */
// include/linux/power_supply.h
// power supply设备描述
struct power_supply_desc {
    // power supply设备名称
	const char *name;
    // power supply设备的类型，电池 UPS 充电器等
	enum power_supply_type type;
	const enum power_supply_usb_type *usb_types;
	size_t num_usb_types;
    // power supply设备属性列表
	const enum power_supply_property *properties;
    // 属性个数
	size_t num_properties;

	/*
	 * Functions for drivers implementing power supply class.
	 * These shouldn't be called directly by other drivers for accessing
	 * this power supply. Instead use power_supply_*() functions (for
	 * example power_supply_get_property()).
	 */
    // power supply驱动需要实现的接口
	int (*get_property)(struct power_supply *psy,
			    enum power_supply_property psp,
			    union power_supply_propval *val);
	int (*set_property)(struct power_supply *psy,
			    enum power_supply_property psp,
			    const union power_supply_propval *val);
	/*
	 * property_is_writeable() will be called during registration
	 * of power supply. If this happens during device probe then it must
	 * not access internal data of device (because probe did not end).
	 */
    // 返回指定的属性值是否可写
	int (*property_is_writeable)(struct power_supply *psy,
				     enum power_supply_property psp);
	void (*external_power_changed)(struct power_supply *psy);
	void (*set_charged)(struct power_supply *psy);

	/*
	 * Set if thermal zone should not be created for this power supply.
	 * For example for virtual supplies forwarding calls to actual
	 * sensors or other supplies.
	 */
	bool no_thermal;
	/* For APM emulation, think legacy userspace. */
	int use_for_apm;
};


// include/linux/power_supply.h
struct power_supply {
    // power supply设备描述
	const struct power_supply_desc *desc;

	char **supplied_to;
	size_t num_supplicants;

	char **supplied_from;
	size_t num_supplies;
	struct device_node *of_node;

	/* Driver private data */
	void *drv_data;

	/* private */
	struct device dev;
    // 用户处理状态改变的work queue, 当power supply设备的状态发生改变，启动一个
    // workqueue，查询并通知所有的supplicants
	struct work_struct changed_work;
	struct delayed_work deferred_register_work;
	spinlock_t changed_lock;
	bool changed;
	bool initialized;
	bool removing;
	atomic_t use_cnt;
#ifdef CONFIG_THERMAL
	struct thermal_zone_device *tzd;
	struct thermal_cooling_device *tcd;
#endif

    // power supply设备LED状态指示
#ifdef CONFIG_LEDS_TRIGGERS
	struct led_trigger *charging_full_trig;
	char *charging_full_trig_name;
	struct led_trigger *charging_trig;
	char *charging_trig_name;
	struct led_trigger *full_trig;
	char *full_trig_name;
	struct led_trigger *online_trig;
	char *online_trig_name;
	struct led_trigger *charging_blink_full_solid_trig;
	char *charging_blink_full_solid_trig_name;
#endif
};
```

power supply设备的类型由`enum power_supply_type`定义，如下：
```c
enum power_supply_type {
    // 未知设备
	POWER_SUPPLY_TYPE_UNKNOWN = 0,
    // 电池供电
	POWER_SUPPLY_TYPE_BATTERY,
    // 不间断供电设备(Uninterruptible Power Supply)
	POWER_SUPPLY_TYPE_UPS,
    // 主供电设备，例如笔记本电源适配器
	POWER_SUPPLY_TYPE_MAINS,
	POWER_SUPPLY_TYPE_USB,			/* Standard Downstream Port */
	POWER_SUPPLY_TYPE_USB_DCP,		/* Dedicated Charging Port */
	POWER_SUPPLY_TYPE_USB_CDP,		/* Charging Downstream Port */
	POWER_SUPPLY_TYPE_USB_ACA,		/* Accessory Charger Adapters */
	POWER_SUPPLY_TYPE_USB_TYPE_C,		/* Type C Port */
	POWER_SUPPLY_TYPE_USB_PD,		/* Power Delivery Port */
	POWER_SUPPLY_TYPE_USB_PD_DRP,		/* PD Dual Role Port */
	POWER_SUPPLY_TYPE_APPLE_BRICK_ID,	/* Apple Charging Method */
	POWER_SUPPLY_TYPE_WIRELESS,		/* Wireless */
};
```

power supply设备属性由`enum power_supply_property`定义，如下：
```c
enum power_supply_property {
	/* Properties of type `int' */
	POWER_SUPPLY_PROP_STATUS = 0,
	POWER_SUPPLY_PROP_CHARGE_TYPE,
	POWER_SUPPLY_PROP_HEALTH,
	POWER_SUPPLY_PROP_PRESENT,
	POWER_SUPPLY_PROP_ONLINE,
	POWER_SUPPLY_PROP_AUTHENTIC,
	POWER_SUPPLY_PROP_TECHNOLOGY,
	POWER_SUPPLY_PROP_CYCLE_COUNT,
	POWER_SUPPLY_PROP_VOLTAGE_MAX,
	POWER_SUPPLY_PROP_VOLTAGE_MIN,
	POWER_SUPPLY_PROP_VOLTAGE_MAX_DESIGN,
	POWER_SUPPLY_PROP_VOLTAGE_MIN_DESIGN,
	POWER_SUPPLY_PROP_VOLTAGE_NOW,
	POWER_SUPPLY_PROP_VOLTAGE_AVG,
	POWER_SUPPLY_PROP_VOLTAGE_OCV,
	POWER_SUPPLY_PROP_VOLTAGE_BOOT,
	POWER_SUPPLY_PROP_CURRENT_MAX,
	POWER_SUPPLY_PROP_CURRENT_NOW,
	POWER_SUPPLY_PROP_CURRENT_AVG,
	POWER_SUPPLY_PROP_CURRENT_BOOT,
	POWER_SUPPLY_PROP_POWER_NOW,
	POWER_SUPPLY_PROP_POWER_AVG,
	POWER_SUPPLY_PROP_CHARGE_FULL_DESIGN,
	POWER_SUPPLY_PROP_CHARGE_EMPTY_DESIGN,
	POWER_SUPPLY_PROP_CHARGE_FULL,
	POWER_SUPPLY_PROP_CHARGE_EMPTY,
	POWER_SUPPLY_PROP_CHARGE_NOW,
	POWER_SUPPLY_PROP_CHARGE_AVG,
	POWER_SUPPLY_PROP_CHARGE_COUNTER,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_CURRENT,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_CURRENT_MAX,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_VOLTAGE,
	POWER_SUPPLY_PROP_CONSTANT_CHARGE_VOLTAGE_MAX,
	POWER_SUPPLY_PROP_CHARGE_CONTROL_LIMIT,
	POWER_SUPPLY_PROP_CHARGE_CONTROL_LIMIT_MAX,
	POWER_SUPPLY_PROP_CHARGE_CONTROL_START_THRESHOLD, /* in percents! */
	POWER_SUPPLY_PROP_CHARGE_CONTROL_END_THRESHOLD, /* in percents! */
	POWER_SUPPLY_PROP_INPUT_CURRENT_LIMIT,
	POWER_SUPPLY_PROP_INPUT_VOLTAGE_LIMIT,
	POWER_SUPPLY_PROP_INPUT_POWER_LIMIT,
	POWER_SUPPLY_PROP_ENERGY_FULL_DESIGN,
	POWER_SUPPLY_PROP_ENERGY_EMPTY_DESIGN,
	POWER_SUPPLY_PROP_ENERGY_FULL,
	POWER_SUPPLY_PROP_ENERGY_EMPTY,
	POWER_SUPPLY_PROP_ENERGY_NOW,
	POWER_SUPPLY_PROP_ENERGY_AVG,
	POWER_SUPPLY_PROP_CAPACITY, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_ALERT_MIN, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_ALERT_MAX, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_ERROR_MARGIN, /* in percents! */
	POWER_SUPPLY_PROP_CAPACITY_LEVEL,
	POWER_SUPPLY_PROP_TEMP,
	POWER_SUPPLY_PROP_TEMP_MAX,
	POWER_SUPPLY_PROP_TEMP_MIN,
	POWER_SUPPLY_PROP_TEMP_ALERT_MIN,
	POWER_SUPPLY_PROP_TEMP_ALERT_MAX,
	POWER_SUPPLY_PROP_TEMP_AMBIENT,
	POWER_SUPPLY_PROP_TEMP_AMBIENT_ALERT_MIN,
	POWER_SUPPLY_PROP_TEMP_AMBIENT_ALERT_MAX,
	POWER_SUPPLY_PROP_TIME_TO_EMPTY_NOW,
	POWER_SUPPLY_PROP_TIME_TO_EMPTY_AVG,
	POWER_SUPPLY_PROP_TIME_TO_FULL_NOW,
	POWER_SUPPLY_PROP_TIME_TO_FULL_AVG,
	POWER_SUPPLY_PROP_TYPE, /* use power_supply.type instead */
	POWER_SUPPLY_PROP_USB_TYPE,
	POWER_SUPPLY_PROP_SCOPE,
	POWER_SUPPLY_PROP_PRECHARGE_CURRENT,
	POWER_SUPPLY_PROP_CHARGE_TERM_CURRENT,
	POWER_SUPPLY_PROP_CALIBRATE,
	POWER_SUPPLY_PROP_MANUFACTURE_YEAR,
	POWER_SUPPLY_PROP_MANUFACTURE_MONTH,
	POWER_SUPPLY_PROP_MANUFACTURE_DAY,
	/* Properties of type `const char *' */
	POWER_SUPPLY_PROP_MODEL_NAME,
	POWER_SUPPLY_PROP_MANUFACTURER,
	POWER_SUPPLY_PROP_SERIAL_NUMBER,
};
```

power supply class向power supply驱动提供统一的驱动编程接口
1. 
