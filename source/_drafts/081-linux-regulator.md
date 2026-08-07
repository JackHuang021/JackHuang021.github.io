---
title: Linux Regulator子系统
tags:
---

## 1. Regulator 介绍

Regulator，中文翻译为“稳压器”，是一种可以自动维持恒定电压（电流）的元器件，Regulator 可以用来为电子设备进行供电，并且可以控其输出开断，或者输出电压（电流）的大小。

Linux Regulator Framework 旨在于提供标准的内核接口，控制系统的 regulators，在系统运行的过程中，动态地改变 regulators 的输出，以达到省电的目的。

## 2. Linux Regulator 软件架构

Linux regulator 软件框架由 4 个部分组成，分别是：machine, regulator driver, consumer driver, Userspace ABI。

### 2.1 Consumer 层

Consumer 指的是由 regulator 供电的设备，Consumer 可以分为两类：
+ Static Consumer：只需要控制关闭或者开启 regulator
+ Dynamic Consumer：除了需要控制 regulator 的开启或关闭，还需要按需动态地调整 regulator 的输出电压（电流）

Linux regulator consumer 层，通过抽象 regulator，为 Consumer 提供操作接口

### 2.2 Regulator Driver

Regulator Driver 需要描述 regulator 的信息，使用 regulator 注册接口对regulator进行注册，注册后会返回一个 `struct regulator_dev` 设备。

### 2.3 Machine

Machine 层用于描述板级的 regulator 层级关系，regulator 的物理限制，包括输出的最大最小值、是否允许关闭，收否在系统启动时进行打开。

### 2.4 Userspace ABI

将 regulator 相关的信息通过sysfs导出到向用户空间，用于监控设备的功耗和状态

## 3. Regulator Driver 实现

一个regulator driver 实现的大致流程如下：
1. 初始化PMIC的寄存器，控制 regulator 的输出最终是需每级调整的电压大小、对应操作的寄存器要通过操作 PMIC 的寄存器来实现的。
2. 填充 regulator 的静态信息（`struct regulator_desc`）和动态信息（`struct regulator_config`），以这两者为参数，调用 regulator 的注册接口，将 regulator 注册到内核中。
3. 提供 regulator_ops 接口，实现对 regulator 的控制。

### 3.1 数据结构

#### 3.1.1 struct regulator_desc

在注册regulator的时候，需要使用 `struct regulator_desc` 结构提供该 regulator 的静态描述。所谓的静态，是指的 PMIC 中 regulators 的物理参数（比如每级调整的电压大小、对应操作的寄存器），这些参数是不会在运行时改变的。

```c
// include/linux/regulator/driver.h
/**
 * struct regulator_desc - Static regulator descriptor
 *
 * Each regulator registered with the core is described with a
 * structure of this type and a struct regulator_config.  This
 * structure contains the non-varying parts of the regulator
 * description.
 *
 * @name: Identifying name for the regulator.
 * @supply_name: Identifying the regulator supply
 * @of_match: Name used to identify regulator in DT.
 * @of_match_full_name: A flag to indicate that the of_match string, if
 *			present, should be matched against the node full_name.
 * @regulators_node: Name of node containing regulator definitions in DT.
 * @of_parse_cb: Optional callback called only if of_match is present.
 *               Will be called for each regulator parsed from DT, during
 *               init_data parsing.
 *               The regulator_config passed as argument to the callback will
 *               be a copy of config passed to regulator_register, valid only
 *               for this particular call. Callback may freely change the
 *               config but it cannot store it for later usage.
 *               Callback should return 0 on success or negative ERRNO
 *               indicating failure.
 * @id: Numerical identifier for the regulator.
 * @ops: Regulator operations table.
 * @irq: Interrupt number for the regulator.
 * @type: Indicates if the regulator is a voltage or current regulator.
 * @owner: Module providing the regulator, used for refcounting.
 *
 * @continuous_voltage_range: Indicates if the regulator can set any
 *                            voltage within constrains range.
 * @n_voltages: Number of selectors available for ops.list_voltage().
 * @n_current_limits: Number of selectors available for current limits
 *
 * @min_uV: Voltage given by the lowest selector (if linear mapping)
 * @uV_step: Voltage increase with each selector (if linear mapping)
 * @linear_min_sel: Minimal selector for starting linear mapping
 * @fixed_uV: Fixed voltage of rails.
 * @ramp_delay: Time to settle down after voltage change (unit: uV/us)
 * @min_dropout_uV: The minimum dropout voltage this regulator can handle
 * @linear_ranges: A constant table of possible voltage ranges.
 * @linear_range_selectors_bitfield: A constant table of voltage range
 *                                   selectors as bitfield values. If
 *                                   pickable ranges are used each range
 *                                   must have corresponding selector here.
 * @n_linear_ranges: Number of entries in the @linear_ranges (and in
 *		     linear_range_selectors_bitfield if used) table(s).
 * @volt_table: Voltage mapping table (if table based mapping)
 * @curr_table: Current limit mapping table (if table based mapping)
 *
 * @vsel_range_reg: Register for range selector when using pickable ranges
 *		    and ``regulator_map_*_voltage_*_pickable`` functions.
 * @vsel_range_mask: Mask for register bitfield used for range selector
 * @range_applied_by_vsel: A flag to indicate that changes to vsel_range_reg
 *			   are only effective after vsel_reg is written
 * @vsel_reg: Register for selector when using ``regulator_map_*_voltage_*``
 * @vsel_mask: Mask for register bitfield used for selector
 * @vsel_step: Specify the resolution of selector stepping when setting
 *	       voltage. If 0, then no stepping is done (requested selector is
 *	       set directly), if >0 then the regulator API will ramp the
 *	       voltage up/down gradually each time increasing/decreasing the
 *	       selector by the specified step value.
 * @csel_reg: Register for current limit selector using regmap set_current_limit
 * @csel_mask: Mask for register bitfield used for current limit selector
 * @apply_reg: Register for initiate voltage change on the output when
 *                using regulator_set_voltage_sel_regmap
 * @apply_bit: Register bitfield used for initiate voltage change on the
 *                output when using regulator_set_voltage_sel_regmap
 * @enable_reg: Register for control when using regmap enable/disable ops
 * @enable_mask: Mask for control when using regmap enable/disable ops
 * @enable_val: Enabling value for control when using regmap enable/disable ops
 * @disable_val: Disabling value for control when using regmap enable/disable ops
 * @enable_is_inverted: A flag to indicate set enable_mask bits to disable
 *                      when using regulator_enable_regmap and friends APIs.
 * @bypass_reg: Register for control when using regmap set_bypass
 * @bypass_mask: Mask for control when using regmap set_bypass
 * @bypass_val_on: Enabling value for control when using regmap set_bypass
 * @bypass_val_off: Disabling value for control when using regmap set_bypass
 * @active_discharge_off: Enabling value for control when using regmap
 *			  set_active_discharge
 * @active_discharge_on: Disabling value for control when using regmap
 *			 set_active_discharge
 * @active_discharge_mask: Mask for control when using regmap
 *			   set_active_discharge
 * @active_discharge_reg: Register for control when using regmap
 *			  set_active_discharge
 * @soft_start_reg: Register for control when using regmap set_soft_start
 * @soft_start_mask: Mask for control when using regmap set_soft_start
 * @soft_start_val_on: Enabling value for control when using regmap
 *                     set_soft_start
 * @pull_down_reg: Register for control when using regmap set_pull_down
 * @pull_down_mask: Mask for control when using regmap set_pull_down
 * @pull_down_val_on: Enabling value for control when using regmap
 *                     set_pull_down
 *
 * @ramp_reg:		Register for controlling the regulator ramp-rate.
 * @ramp_mask:		Bitmask for the ramp-rate control register.
 * @ramp_delay_table:	Table for mapping the regulator ramp-rate values. Values
 *			should be given in units of V/S (uV/uS). See the
 *			regulator_set_ramp_delay_regmap().
 * @n_ramp_values:	number of elements at @ramp_delay_table.
 *
 * @enable_time: Time taken for initial enable of regulator (in uS).
 * @off_on_delay: guard time (in uS), before re-enabling a regulator
 *
 * @poll_enabled_time: The polling interval (in uS) to use while checking that
 *                     the regulator was actually enabled. Max upto enable_time.
 *
 * @of_map_mode: Maps a hardware mode defined in a DeviceTree to a standard mode
 */
struct regulator_desc {
	/*  该 regulator 的名称 */
	const char *name;
	/* 上级 regulator 的名称 */
	const char *supply_name;
	/* 提供设备树节点信息，以便在注册的时候自动从DTS中解析init_data */
	const char *of_match;
	bool of_match_full_name;
	const char *regulators_node;
	int (*of_parse_cb)(struct device_node *,
			    const struct regulator_desc *,
			    struct regulator_config *);
	int id;
	unsigned int continuous_voltage_range:1;
	/* 该 regulator 可以输出的电压值个数 */
	unsigned n_voltages;
	unsigned int n_current_limits;
	/* regulator 的操作函数集 */
	const struct regulator_ops *ops;
	int irq;
	/* regulator 的类型，电压和电流两种 */
	enum regulator_type type;
	struct module *owner;
	/* 可以输出的最小电压 */
	unsigned int min_uV;
	/* 每级可调整的电压大小 */
	unsigned int uV_step;
	unsigned int linear_min_sel;
	int fixed_uV;
	unsigned int ramp_delay;
	int min_dropout_uV;

	const struct linear_range *linear_ranges;
	const unsigned int *linear_range_selectors_bitfield;

	int n_linear_ranges;

	const unsigned int *volt_table;
	const unsigned int *curr_table;

	unsigned int vsel_range_reg;
	unsigned int vsel_range_mask;
	bool range_applied_by_vsel;
	unsigned int vsel_reg;
	unsigned int vsel_mask;
	unsigned int vsel_step;
	unsigned int csel_reg;
	unsigned int csel_mask;
	unsigned int apply_reg;
	unsigned int apply_bit;
	unsigned int enable_reg;
	unsigned int enable_mask;
	unsigned int enable_val;
	unsigned int disable_val;
	bool enable_is_inverted;
	unsigned int bypass_reg;
	unsigned int bypass_mask;
	unsigned int bypass_val_on;
	unsigned int bypass_val_off;
	unsigned int active_discharge_on;
	unsigned int active_discharge_off;
	unsigned int active_discharge_mask;
	unsigned int active_discharge_reg;
	unsigned int soft_start_reg;
	unsigned int soft_start_mask;
	unsigned int soft_start_val_on;
	unsigned int pull_down_reg;
	unsigned int pull_down_mask;
	unsigned int pull_down_val_on;
	unsigned int ramp_reg;
	unsigned int ramp_mask;
	const unsigned int *ramp_delay_table;
	unsigned int n_ramp_values;

	unsigned int enable_time;

	unsigned int off_on_delay;

	unsigned int poll_enabled_time;

	unsigned int (*of_map_mode)(unsigned int mode);
};
```

RK808 其中的一个 regulator 的静态描述如下

```c
// drivers/regulator/rk808-regulator.c
static const struct regulator_desc rk808_reg[] = {
	{
		.name = "DCDC_REG1",
		.supply_name = "vcc1",
		.of_match = of_match_ptr("DCDC_REG1"),
		.regulators_node = of_match_ptr("regulators"),
		.id = RK808_ID_DCDC1,
		.ops = &rk808_buck1_2_ops,
		.type = REGULATOR_VOLTAGE,
		.min_uV = 712500,
		.uV_step = 12500,
		.n_voltages = 64,
		.vsel_reg = RK808_BUCK1_ON_VSEL_REG,
		.vsel_mask = RK808_BUCK_VSEL_MASK,
		.enable_reg = RK808_DCDC_EN_REG,
		.enable_mask = BIT(0),
		.ramp_reg = RK808_BUCK1_CONFIG_REG,
		.ramp_mask = RK808_RAMP_RATE_MASK,
		.ramp_delay_table = rk808_buck1_2_ramp_table,
		.n_ramp_values = ARRAY_SIZE(rk808_buck1_2_ramp_table),
		.owner = THIS_MODULE,
	}, {
	...
	};
```

#### 3.1.2 struct regulator_config

```c
// include/linux/regulator/driver.h
/**
 * struct regulator_config - Dynamic regulator descriptor
 *
 * Each regulator registered with the core is described with a
 * structure of this type and a struct regulator_desc.  This structure
 * contains the runtime variable parts of the regulator description.
 *
 * @dev: struct device for the regulator
 * @init_data: platform provided init data, passed through by driver
 * @driver_data: private regulator data
 * @of_node: OpenFirmware node to parse for device tree bindings (may be
 *           NULL).
 * @regmap: regmap to use for core regmap helpers if dev_get_regmap() is
 *          insufficient.
 * @ena_gpiod: GPIO controlling regulator enable.
 */
struct regulator_config {
	struct device *dev;
	const struct regulator_init_data *init_data;
	void *driver_data;
	struct device_node *of_node;
	struct regmap *regmap;
	/* 控制regulator使能的GPIO及其active极性 */
	struct gpio_desc *ena_gpiod;
};
```

#### 3.1.3 struct regulation_constraints

`struct regulator_constraints` 用于描述 regulator 约束，保存 regulator 的物理限制

```c
// include/linux/regulator/machine.h
/**
 * struct regulation_constraints - regulator operating constraints.
 *
 * This struct describes regulator and board/machine specific constraints.
 *
 * @name: Descriptive name for the constraints, used for display purposes.
 *
 * @min_uV: Smallest voltage consumers may set.
 * @max_uV: Largest voltage consumers may set.
 * @uV_offset: Offset applied to voltages from consumer to compensate for
 *             voltage drops.
 *
 * @min_uA: Smallest current consumers may set.
 * @max_uA: Largest current consumers may set.
 * @ilim_uA: Maximum input current.
 * @system_load: Load that isn't captured by any consumer requests.
 *
 * @over_curr_limits:		Limits for acting on over current.
 * @over_voltage_limits:	Limits for acting on over voltage.
 * @under_voltage_limits:	Limits for acting on under voltage.
 * @temp_limits:		Limits for acting on over temperature.
 *
 * @max_spread: Max possible spread between coupled regulators
 * @max_uV_step: Max possible step change in voltage
 * @valid_modes_mask: Mask of modes which may be configured by consumers.
 * @valid_ops_mask: Operations which may be performed by consumers.
 *
 * @always_on: Set if the regulator should never be disabled.
 * @boot_on: Set if the regulator is enabled when the system is initially
 *           started.  If the regulator is not enabled by the hardware or
 *           bootloader then it will be enabled when the constraints are
 *           applied.
 * @apply_uV: Apply the voltage constraint when initialising.
 * @ramp_disable: Disable ramp delay when initialising or when setting voltage.
 * @soft_start: Enable soft start so that voltage ramps slowly.
 * @pull_down: Enable pull down when regulator is disabled.
 * @over_current_protection: Auto disable on over current event.
 *
 * @over_current_detection: Configure over current limits.
 * @over_voltage_detection: Configure over voltage limits.
 * @under_voltage_detection: Configure under voltage limits.
 * @over_temp_detection: Configure over temperature limits.
 *
 * @input_uV: Input voltage for regulator when supplied by another regulator.
 *
 * @state_disk: State for regulator when system is suspended in disk mode.
 * @state_mem: State for regulator when system is suspended in mem mode.
 * @state_standby: State for regulator when system is suspended in standby
 *                 mode.
 * @initial_state: Suspend state to set by default.
 * @initial_mode: Mode to set at startup.
 * @ramp_delay: Time to settle down after voltage change (unit: uV/us)
 * @settling_time: Time to settle down after voltage change when voltage
 *		   change is non-linear (unit: microseconds).
 * @settling_time_up: Time to settle down after voltage increase when voltage
 *		      change is non-linear (unit: microseconds).
 * @settling_time_down : Time to settle down after voltage decrease when
 *			 voltage change is non-linear (unit: microseconds).
 * @active_discharge: Enable/disable active discharge. The enum
 *		      regulator_active_discharge values are used for
 *		      initialisation.
 * @enable_time: Turn-on time of the rails (unit: microseconds)
 */
struct regulation_constraints {

	/* regulator 约束的名称 */
	const char *name;

	/* voltage output range (inclusive) - for voltage control */
	/* regulator 最小、最大输出电压 */
	int min_uV;
	int max_uV;

	int uV_offset;

	/* current output range (inclusive) - for current control */
	int min_uA;
	int max_uA;
	int ilim_uA;

	int system_load;

	/* used for coupled regulators */
	u32 *max_spread;

	/* used for changing voltage in steps */
	int max_uV_step;

	/* valid regulator operating modes for this machine */
	/* regulator 支持的操作模式，基本上都是 Normal 模式 */
	unsigned int valid_modes_mask;

	/* valid operations for regulator on this machine */
	unsigned int valid_ops_mask;

	/* regulator input voltage - only if supply is another regulator */
	/* 输入电压的大小 */
	int input_uV;

	/* regulator suspend states for global PMIC STANDBY/HIBERNATE */
	struct regulator_state state_disk;
	struct regulator_state state_mem;
	struct regulator_state state_standby;
	struct notification_limit over_curr_limits;
	struct notification_limit over_voltage_limits;
	struct notification_limit under_voltage_limits;
	struct notification_limit temp_limits;
	suspend_state_t initial_state; /* suspend state to set at init */

	/* mode to set on startup */
	unsigned int initial_mode;

	unsigned int ramp_delay;
	unsigned int settling_time;
	unsigned int settling_time_up;
	unsigned int settling_time_down;
	unsigned int enable_time;

	unsigned int active_discharge;

	/* constraint flags */
	/* regulator 是否一直保持开启 */
	unsigned always_on:1;	/* regulator never off when system is on */
	/* 是否在系统启动时则开启该 regulator */
	unsigned boot_on:1;	/* bootloader/firmware enabled regulator */
	unsigned apply_uV:1;	/* apply uV constraint if min == max */
	unsigned ramp_disable:1; /* disable ramp delay */
	unsigned soft_start:1;	/* ramp voltage slowly */
	unsigned pull_down:1;	/* pull down resistor when regulator off */
	unsigned over_current_protection:1; /* auto disable on over current */
	unsigned over_current_detection:1; /* notify on over current */
	unsigned over_voltage_detection:1; /* notify on over voltage */
	unsigned under_voltage_detection:1; /* notify on under voltage */
	unsigned over_temp_detection:1; /* notify on over temperature */
};
```

#### 3.1.4 struct regulator_ops

`struct regulator_ops` 抽象了对 regulator 的操作接口，由 regulator driver 实现

```c
// include/linux/regulator/driver.h
/**
 * struct regulator_ops - regulator operations.
 *
 * @enable: Configure the regulator as enabled.
 * @disable: Configure the regulator as disabled.
 * @is_enabled: Return 1 if the regulator is enabled, 0 if not.
 *		May also return negative errno.
 *
 * @set_voltage: Set the voltage for the regulator within the range specified.
 *               The driver should select the voltage closest to min_uV.
 * @set_voltage_sel: Set the voltage for the regulator using the specified
 *                   selector.
 * @map_voltage: Convert a voltage into a selector
 * @get_voltage: Return the currently configured voltage for the regulator;
 *                   return -ENOTRECOVERABLE if regulator can't be read at
 *                   bootup and hasn't been set yet.
 * @get_voltage_sel: Return the currently configured voltage selector for the
 *                   regulator; return -ENOTRECOVERABLE if regulator can't
 *                   be read at bootup and hasn't been set yet.
 * @list_voltage: Return one of the supported voltages, in microvolts; zero
 *	if the selector indicates a voltage that is unusable on this system;
 *	or negative errno.  Selectors range from zero to one less than
 *	regulator_desc.n_voltages.  Voltages may be reported in any order.
 *
 * @set_current_limit: Configure a limit for a current-limited regulator.
 *                     The driver should select the current closest to max_uA.
 * @get_current_limit: Get the configured limit for a current-limited regulator.
 * @set_input_current_limit: Configure an input limit.
 *
 * @set_over_current_protection: Support enabling of and setting limits for over
 *	current situation detection. Detection can be configured for three
 *	levels of severity.
 *
 *	- REGULATOR_SEVERITY_PROT should automatically shut down the regulator(s).
 *
 *	- REGULATOR_SEVERITY_ERR should indicate that over-current situation is
 *		  caused by an unrecoverable error but HW does not perform
 *		  automatic shut down.
 *
 *	- REGULATOR_SEVERITY_WARN should indicate situation where hardware is
 *		  still believed to not be damaged but that a board sepcific
 *		  recovery action is needed. If lim_uA is 0 the limit should not
 *		  be changed but the detection should just be enabled/disabled as
 *		  is requested.
 *
 * @set_over_voltage_protection: Support enabling of and setting limits for over
 *	voltage situation detection. Detection can be configured for same
 *	severities as over current protection. Units of uV.
 * @set_under_voltage_protection: Support enabling of and setting limits for
 *	under voltage situation detection. Detection can be configured for same
 *	severities as over current protection. Units of uV.
 * @set_thermal_protection: Support enabling of and setting limits for over
 *	temperature situation detection.Detection can be configured for same
 *	severities as over current protection. Units of degree Kelvin.
 *
 * @set_active_discharge: Set active discharge enable/disable of regulators.
 *
 * @set_mode: Set the configured operating mode for the regulator.
 * @get_mode: Get the configured operating mode for the regulator.
 * @get_error_flags: Get the current error(s) for the regulator.
 * @get_status: Return actual (not as-configured) status of regulator, as a
 *	REGULATOR_STATUS value (or negative errno)
 * @get_optimum_mode: Get the most efficient operating mode for the regulator
 *                    when running with the specified parameters.
 * @set_load: Set the load for the regulator.
 *
 * @set_bypass: Set the regulator in bypass mode.
 * @get_bypass: Get the regulator bypass mode state.
 *
 * @enable_time: Time taken for the regulator voltage output voltage to
 *               stabilise after being enabled, in microseconds.
 * @set_ramp_delay: Set the ramp delay for the regulator. The driver should
 *		select ramp delay equal to or less than(closest) ramp_delay.
 * @set_voltage_time: Time taken for the regulator voltage output voltage
 *               to stabilise after being set to a new value, in microseconds.
 *               The function receives the from and to voltage as input, it
 *               should return the worst case.
 * @set_voltage_time_sel: Time taken for the regulator voltage output voltage
 *               to stabilise after being set to a new value, in microseconds.
 *               The function receives the from and to voltage selector as
 *               input, it should return the worst case.
 * @set_soft_start: Enable soft start for the regulator.
 *
 * @set_suspend_voltage: Set the voltage for the regulator when the system
 *                       is suspended.
 * @set_suspend_enable: Mark the regulator as enabled when the system is
 *                      suspended.
 * @set_suspend_disable: Mark the regulator as disabled when the system is
 *                       suspended.
 * @set_suspend_mode: Set the operating mode for the regulator when the
 *                    system is suspended.
 * @resume: Resume operation of suspended regulator.
 * @set_pull_down: Configure the regulator to pull down when the regulator
 *		   is disabled.
 *
 * This struct describes regulator operations which can be implemented by
 * regulator chip drivers.
 */
struct regulator_ops {

	/* enumerate supported voltages */
	int (*list_voltage) (struct regulator_dev *, unsigned selector);

	/* get/set regulator voltage */
	int (*set_voltage) (struct regulator_dev *, int min_uV, int max_uV,
			    unsigned *selector);
	int (*map_voltage)(struct regulator_dev *, int min_uV, int max_uV);
	int (*set_voltage_sel) (struct regulator_dev *, unsigned selector);
	int (*get_voltage) (struct regulator_dev *);
	int (*get_voltage_sel) (struct regulator_dev *);

	/* get/set regulator current  */
	int (*set_current_limit) (struct regulator_dev *,
				 int min_uA, int max_uA);
	int (*get_current_limit) (struct regulator_dev *);

	int (*set_input_current_limit) (struct regulator_dev *, int lim_uA);
	int (*set_over_current_protection)(struct regulator_dev *, int lim_uA,
					   int severity, bool enable);
	int (*set_over_voltage_protection)(struct regulator_dev *, int lim_uV,
					   int severity, bool enable);
	int (*set_under_voltage_protection)(struct regulator_dev *, int lim_uV,
					    int severity, bool enable);
	int (*set_thermal_protection)(struct regulator_dev *, int lim,
				      int severity, bool enable);
	int (*set_active_discharge)(struct regulator_dev *, bool enable);

	/* enable/disable regulator */
	int (*enable) (struct regulator_dev *);
	int (*disable) (struct regulator_dev *);
	int (*is_enabled) (struct regulator_dev *);

	/* get/set regulator operating mode (defined in consumer.h) */
	int (*set_mode) (struct regulator_dev *, unsigned int mode);
	unsigned int (*get_mode) (struct regulator_dev *);

	/* retrieve current error flags on the regulator */
	int (*get_error_flags)(struct regulator_dev *, unsigned int *flags);

	/* Time taken to enable or set voltage on the regulator */
	int (*enable_time) (struct regulator_dev *);
	int (*set_ramp_delay) (struct regulator_dev *, int ramp_delay);
	int (*set_voltage_time) (struct regulator_dev *, int old_uV,
				 int new_uV);
	int (*set_voltage_time_sel) (struct regulator_dev *,
				     unsigned int old_selector,
				     unsigned int new_selector);

	int (*set_soft_start) (struct regulator_dev *);

	/* report regulator status ... most other accessors report
	 * control inputs, this reports results of combining inputs
	 * from Linux (and other sources) with the actual load.
	 * returns REGULATOR_STATUS_* or negative errno.
	 */
	int (*get_status)(struct regulator_dev *);

	/* get most efficient regulator operating mode for load */
	unsigned int (*get_optimum_mode) (struct regulator_dev *, int input_uV,
					  int output_uV, int load_uA);
	/* set the load on the regulator */
	int (*set_load)(struct regulator_dev *, int load_uA);

	/* control and report on bypass mode */
	int (*set_bypass)(struct regulator_dev *dev, bool enable);
	int (*get_bypass)(struct regulator_dev *dev, bool *enable);

	/* the operations below are for configuration of regulator state when
	 * its parent PMIC enters a global STANDBY/HIBERNATE state */

	/* set regulator suspend voltage */
	int (*set_suspend_voltage) (struct regulator_dev *, int uV);

	/* enable/disable regulator in suspend state */
	int (*set_suspend_enable) (struct regulator_dev *);
	int (*set_suspend_disable) (struct regulator_dev *);

	/* set regulator suspend operating mode (defined in consumer.h) */
	int (*set_suspend_mode) (struct regulator_dev *, unsigned int mode);

	int (*resume)(struct regulator_dev *rdev);

	int (*set_pull_down) (struct regulator_dev *);
};
```

#### 3.1.5 struct regulator_dev

`struct regulator_dev` 是regulator设备的抽象，调用`regulator_register()`将regulator 注册到 kernel 之后，regulator 就会分配一个 `struct regulator_dev` 变量，后续所有的 regulator 操作，都将以该变量为对象。

```c
/*
 * struct regulator_dev
 *
 * Voltage / Current regulator class device. One for each
 * regulator.
 *
 * This should *not* be used directly by anything except the regulator
 * core and notification injection (which should take the mutex and do
 * no other direct access).
 */
struct regulator_dev {
	/* 指向 regulator 的静态信息 */
	const struct regulator_desc *desc;
	/* 表示该 regulator 已经被 某个consumer独占了 */
	int exclusive;
	u32 use_count;
	u32 open_count;
	u32 bypass_count;

	/* lists we belong to */
	struct list_head list; /* list of all regulators */

	/* lists we own */
	struct list_head consumer_list; /* consumers we supply */

	struct coupling_desc coupling_desc;

	struct blocking_notifier_head notifier;
	struct ww_mutex mutex; /* consumer lock */
	struct task_struct *mutex_owner;
	int ref_cnt;
	struct module *owner;
	struct device dev;
	/* 保存 regulator 的约束信息 */
	struct regulation_constraints *constraints;
	struct regulator *supply;	/* for tree */
	const char *supply_name;
	struct regmap *regmap;

	struct delayed_work disable_work;
 
	void *reg_data;		/* regulator_dev data */

	struct dentry *debugfs;

	struct regulator_enable_gpio *ena_pin;
	unsigned int ena_gpio_state:1;

	unsigned int is_switch:1;

	/* time when this regulator was disabled last time */
	ktime_t last_off;
	int cached_err;
	bool use_cached_err;
	spinlock_t err_lock;
};
```

### 3.2 Regulator Core 初始化过程

regulator core 的初始化操作由 `regulator_init()` 接口负责，主要工作包括：
+ 注册 regulator class (/sys/class/regulator)
+ 注册用于调试的 debugfs (/sys/kernel/debug/regulator)

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250411095701.png)

### 3.3 Regulator 注册过程

regulator 的注册，由 `regulator_register() / devm_regulator_register()` 接口负责。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250411102709.png)

## 4. 瑞芯微 RK808 regulator 驱动

### 4.1 RK808 介绍

RK808 是⼀款⾼性能 PMIC，RK808 集成 4 个⼤电流 DCDC、8 个 LDO、2个开关SWITCH、1 个 RTC、可调上电时序等功能

### 4.2 分析 regulator 硬件拓扑

编写 Regulator 驱动，首先需要弄清楚硬件的 regulator 层级关系，以 NanoPC-T4 开发板（使用RK808作为PMIC）为例，其 regulator 的拓扑结构如下:

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250411094805.png)

接下来根据实际的硬件拓扑，在 设备树 中描述 regulator 的拓扑结构，并在系统初始化的时候，由 regulator driver 将这些 regulator 注册到内核中，下面是 RK808 在设备树里面的描述：

```c
rk808: pmic@1b {
	compatible = "rockchip,rk808";
	reg = <0x1b>;
	clock-output-names = "xin32k", "rtc_clko_wifi";
	#clock-cells = <1>;
	interrupt-parent = <&gpio1>;
	interrupts = <21 IRQ_TYPE_LEVEL_LOW>;
	pinctrl-names = "default";
	pinctrl-0 = <&pmic_int_l>, <&ap_pwroff>, <&clk_32k>;
	rockchip,system-power-controller;
	wakeup-source;

	vcc1-supply = <&vcc3v3_sys>;
	vcc2-supply = <&vcc3v3_sys>;
	vcc3-supply = <&vcc3v3_sys>;
	vcc4-supply = <&vcc3v3_sys>;
	vcc6-supply = <&vcc3v3_sys>;
	vcc7-supply = <&vcc3v3_sys>;
	vcc8-supply = <&vcc3v3_sys>;
	vcc9-supply = <&vcc3v3_sys>;
	vcc10-supply = <&vcc3v3_sys>;
	vcc11-supply = <&vcc3v3_sys>;
	vcc12-supply = <&vcc3v3_sys>;
	vddio-supply = <&vcc_3v0>;

	regulators {
		vdd_center: DCDC_REG1 {
			regulator-always-on;
			regulator-boot-on;
			regulator-min-microvolt = <750000>;
			regulator-max-microvolt = <1350000>;
			regulator-name = "vdd_center";
			regulator-ramp-delay = <6001>;

			regulator-state-mem {
				regulator-off-in-suspend;
			};
		};
	...
};
```

### 4.3 RK808 regulator driver 实现

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250411095205.png)

## 5. Consumer 如何使用 regulator

regulator framework 向 consumer 提供的接口声明位于 `include/linux/regulator/consumer.h` 中，源码实现位于`drivers/regulator/core.c`，包括 regulator 的获取、使能、修改等接口

### 5.1 相关数据结构

#### 5.1.1 struct regulator

用于从 Consumer 的角度抽象一个 regulator，这个 regulator 供 Consumer 使用。

```c
// drivers/regulator/internal.h
/*
 * struct regulator
 *
 * One for each consumer device.
 * @voltage - a voltage array for each state of runtime, i.e.:
 *            PM_SUSPEND_ON
 *            PM_SUSPEND_TO_IDLE
 *            PM_SUSPEND_STANDBY
 *            PM_SUSPEND_MEM
 *            PM_SUSPEND_MAX
 */
struct regulator {
	/* consumer 对应的 kernel device */
	struct device *dev;
	/* consumer 对应的 regulator 会以链表形式添加到 rdev->consumer_list */
	struct list_head list;
	/* 是否需要 regulator 一直打开 */
	unsigned int always_on:1;
	unsigned int bypass:1;
	unsigned int device_link:1;
	int uA_load;
	unsigned int enable_count;
	unsigned int deferred_disables;
	struct regulator_voltage voltage[REGULATOR_STATES_NUM];
	/* 对应的 regulator 的名字 */
	const char *supply_name;
	struct device_attribute dev_attr;
	/* 对应的 regulator 设备 */
	struct regulator_dev *rdev;
	struct dentry *debugfs;
};
```

### 5.2 Consumer 层接口介绍

#### 5.2.1 Regulator get() put() 接口

regulator get() 的类型可以分为 独占、非独占，consumer 使用 独占 get() 接口获取 regulator，则会将 `regulator->exclusive` 置1，其余的 consumer 无法再获取该 regulator。

```c
// include/linux/regulator/consumer.h
/* regulator get and put */
struct regulator *__must_check regulator_get(struct device *dev,
					     const char *id);
struct regulator *__must_check devm_regulator_get(struct device *dev,
					     const char *id);

/* 独占 get() 接口 */
struct regulator *__must_check regulator_get_exclusive(struct device *dev,
						       const char *id);
struct regulator *__must_check devm_regulator_get_exclusive(struct device *dev,
							const char *id);
struct regulator *__must_check regulator_get_optional(struct device *dev,
						      const char *id);
struct regulator *__must_check devm_regulator_get_optional(struct device *dev,
							   const char *id);
int devm_regulator_get_enable(struct device *dev, const char *id);
int devm_regulator_get_enable_optional(struct device *dev, const char *id);
void regulator_put(struct regulator *regulator);
void devm_regulator_put(struct regulator *regulator);
```

regulator 的 get 过程如下：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250411114137.png)

#### 5.2.2 Regulator 控制、状态获取接口

控制有关的包括：控制regulator 输出使能，电压、电流设置；

状态获取包括：输出开关状态；是否可以改变电压；支持的电压列表；当前输出电压；当前电流限制；当前mode；等等。

```c
// include/linux/regulator/consumer.h
/* regulator output control and status */
/* Regulator 输出使能控制接口 */
int __must_check regulator_enable(struct regulator *regulator);
int regulator_disable(struct regulator *regulator);
int regulator_force_disable(struct regulator *regulator);
int regulator_is_enabled(struct regulator *regulator);
int regulator_disable_deferred(struct regulator *regulator, int ms);

/* 电压 电流设置接口 */
int regulator_count_voltages(struct regulator *regulator);
int regulator_list_voltage(struct regulator *regulator, unsigned selector);
int regulator_is_supported_voltage(struct regulator *regulator,
				   int min_uV, int max_uV);
unsigned int regulator_get_linear_step(struct regulator *regulator);
int regulator_set_voltage(struct regulator *regulator, int min_uV, int max_uV);
int regulator_set_voltage_time(struct regulator *regulator,
			       int old_uV, int new_uV);
int regulator_get_voltage(struct regulator *regulator);
int regulator_sync_voltage(struct regulator *regulator);
int regulator_set_current_limit(struct regulator *regulator,
			       int min_uA, int max_uA);
int regulator_get_current_limit(struct regulator *regulator);

int regulator_set_mode(struct regulator *regulator, unsigned int mode);
unsigned int regulator_get_mode(struct regulator *regulator);
int regulator_get_error_flags(struct regulator *regulator,
				unsigned int *flags);
int regulator_set_load(struct regulator *regulator, int load_uA);

int regulator_allow_bypass(struct regulator *regulator, bool allow);
```

#### 5.2.3 Notifier 相关的接口

Consumer 可以使用 notifier 接口，注册一个回调函数，当 regulator 状态发生变化时，来进行相关的处理。

```c
// include/linux/regulator/consumer.h
/* regulator notifier block */
int regulator_register_notifier(struct regulator *regulator,
			      struct notifier_block *nb);
int devm_regulator_register_notifier(struct regulator *regulator,
				     struct notifier_block *nb);
int regulator_unregister_notifier(struct regulator *regulator,
				struct notifier_block *nb);
void devm_regulator_unregister_notifier(struct regulator *regulator,
					struct notifier_block *nb);
```

### 5.3 Consumer 使用 regulator 示例

Consumer 通过 regulator consumer 层提供的接口来对 regulator 进行操作。

1. 在设备树中描述使用到的 regulator
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250411093534.png)

2. 使用regulator consumer 层提供的接口来对 regulator 进行操作。

```c
/**
 * mmc_regulator_get_supply - try to get VMMC and VQMMC regulators for a host
 * @mmc: the host to regulate
 *
 * Returns 0 or errno. errno should be handled, it is either a critical error
 * or -EPROBE_DEFER. 0 means no critical error but it does not mean all
 * regulators have been found because they all are optional. If you require
 * certain regulators, you need to check separately in your driver if they got
 * populated after calling this function.
 */
int mmc_regulator_get_supply(struct mmc_host *mmc)
{
	struct device *dev = mmc_dev(mmc);
	int ret;

	mmc->supply.vmmc = devm_regulator_get_optional(dev, "vmmc");
	mmc->supply.vqmmc = devm_regulator_get_optional(dev, "vqmmc");

	if (IS_ERR(mmc->supply.vmmc)) {
		if (PTR_ERR(mmc->supply.vmmc) == -EPROBE_DEFER)
			return -EPROBE_DEFER;
		dev_dbg(dev, "No vmmc regulator found\n");
	} else {
		ret = mmc_regulator_get_ocrmask(mmc->supply.vmmc);
		if (ret > 0)
			mmc->ocr_avail = ret;
		else
			dev_warn(dev, "Failed getting OCR mask: %d\n", ret);
	}

	if (IS_ERR(mmc->supply.vqmmc)) {
		if (PTR_ERR(mmc->supply.vqmmc) == -EPROBE_DEFER)
			return -EPROBE_DEFER;
		dev_dbg(dev, "No vqmmc regulator found\n");
	}

	return 0;
}
EXPORT_SYMBOL_GPL(mmc_regulator_get_supply);
```

## 6. 参考资料

1. [http://www.wowotech.net/pm_subsystem/regulator_framework_overview.html](http://www.wowotech.net/pm_subsystem/regulator_framework_overview.html)
2. [https://origin.kernel.org/doc/html/latest/power/regulator/](https://origin.kernel.org/doc/html/latest/power/regulator/)
