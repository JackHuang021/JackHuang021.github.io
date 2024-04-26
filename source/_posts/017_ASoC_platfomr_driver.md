---
title: ASoC Platform Driver
tags:
  - Linux
  - ASoC
categories:
  - Linux
abbrlink: 131a876a
date: 2022-10-26 13:42:24
---

### 概述
ALSA是Advanced Linux Sound Architecture的缩写，[ALSA官网地址](https://www.alsa-project.org/)
<!-- more -->

ALSA作为Linux现在主流的音频体系架构，提供了内核的驱动框架，也提供了应用层的
`alsa-lib`库，ALSA还提供了`alsa-utils`应用程序，方便进行音频控制

音频设备文件结构如下

![](https://raw.githubusercontent.com/JackHuang021/images/master/20221108112754.png)
+ controlC0: 用于card0声卡的控制
+ pcmC0D0c: 用于card0 device0录音的pcm设备
+ pcmC0D0p: 用于card0 device0播放的pcm设备
+ timer: 定时器

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240124105117.png)


### ASoC驱动框架
ASoC（ALSA system on chip）是建立在标准ALSA驱动层上，对底层的ALSA框架封装了一层，为了更好的支持嵌入式CPU和音频编解码设备的一套软件体系，ASoC驱动主要由platform驱动、codec驱动、machine驱动组成。

machine驱动：充当描述和绑定其他组件驱动程序以形成ALSA声卡的粘合剂，machine可以理解为对声卡的抽象，它把cpu_dai，codec_dai通过dai_link链接起来，然后注册snd_soc_card。该驱动实现`struct snd_soc_card`的定义和注册，并通过指定`struct snd_soc_dai_link`中的`codec_name`, `platform_name`, `codec_dai_name`, `platform_dai_name`从而实现与其他各个驱动组件的绑定

platform驱动：一般指某一个SoC平台，与音频相关的通常包括SoC中的时钟、DMA、I2S、I2C等，我们可以认为他们组成了一个对应的音频platform，这个Platform只与SoC有关，这样我们可以把Platform抽象出来

codec驱动：codec字面上的意思就是编解码器，codec和platform一样，可以被多个machine使用，嵌入式codec一般通过I2C或SPI对内部的寄存器进行控制

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240124105147.png)

### 相关名词描述

DAPM（Dynamic Audio Power Management）：动态音频电源管理，用于Linux设备使用音频子系统内的最低电量，它独立于其他内核PM，DAPM对所有的用户空间应用程序来说也是完全透明的，因为所有电源切换都是在ASoC核心内完成的，对于用户空间应用程序，不需要更改代码，DAPM根据当前激活的音频流和声卡中的Mixer等的配置来决定哪些音频控件的电源开关打开或关闭

DAI(Digital Audio Interface)数字音频接口：数字音频接口全部是硬件接口，即同一个主板上IC芯片和IC芯片之间的通讯协议，数字音频接口有PCM、I2S、AC97、PDM

PCM编码（Pulse Code Modulation）：通过等时间间隔（即采样率时钟周期）采样将模拟信号转化为数字信号的方法，PCM使用等间隔采样方法，将每次采样的模拟分量幅度表示为N位的数字分量，因此PCM方式每次采样的结果都是N bit长的数据

Kcontrols：代表声卡里面的各种硬件开关，滑动控件等，通过软件定义kcontrol可以通过用户态配置硬件寄存器的开关，ALSA用snd_kcontrol_new定义kcontrol

Widget：具备路径和电源管理的kcontrol

Mixer：多个输入混合成一个输出，可以使用SND_SOC_DAPM_MIXER来定义这种类型的widget，mixer的widget类型为snd_soc_dapm_mixer

Mux：多路选择器，多路输入，但是只能选择一路作为输出，可以使用SND_SOC_DAPM_MUX来定义这个widget，widget类型为snd_soc_dapm_mux

pga：单路输入，单路输出，带gain调整的部件，可以使用SND_SOC_DAPM_PGA来定义这个widget，所属widget类型为snd_soc_dapm_pga

Path：path相当于电路中的一条跳线，它把一个widget的输出端和另一个widget的输入端连接在一起

Route：route用来描述一条完整的路径，它包括 起始端widget -> path的输入 -> path的输出 -> 目标端widget，DAPM使用`struct snd_soc_dapm_route`结构来描述这样一个完整路径

### Control设备和kcontrol

#### Control设备
Control是音频驱动中用来表示用户可操作的音频参数或功能的抽象设备，它可以是音量控制、Mixer（混音控制）、Mux（开关控制）等，Control提供了一个统一的接口，用于能够通过音频设备驱动程序来管理和调整音频参数，ALSA core层已经实现了Control中间层，在`include\sound\control.h`中定义了所有的Control API

##### Control设备的创建
Control设备和PCM设备一样，都属于声卡下的逻辑设备。用户空间的应用程序通过alsa-lib访问该Control设备，读取或控制控件的控制状态，从而达到控制音频Codec进行各种Mixer等控制操作。

在声卡的初始化过程中会调用到`snd_ctrl_create()`来初始化Control设备，具体的流程如下：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240410140458.png)

#### kcontrol
kcontrol是一种控件，也可以理解为switch，主要实现控制声卡的音量、混音等一系列控制

kcontrol对应的数据结构是`struct snd_kcontrol_new`和`struct kcontrol`，kcontol的创建步骤如下：
+ 在codec驱动中定义`struct snd_kcontrol_new`数组
+ 在声卡初始化的阶段，通过`snd_soc_component_controls()`创建并添加多个kcontrol到`struct snd_card`的controls链表中

##### kcontrol相关的数据结构
`struct snd_kcontrol_new`结构体
```c
// include/sound/control.h
struct snd_kcontrol_new {
	snd_ctl_elem_iface_t iface;	/* interface identifier */
	unsigned int device;		/* device/client number */
	unsigned int subdevice;		/* subdevice (substream) number */
	const char *name;		/* ASCII name of item */
	unsigned int index;		/* index of item */
	/* 控件的访问类型 */
	unsigned int access;		/* access rights */
	unsigned int count;		/* count of same elements */
	/* info回调函数用于获取控件的详细信息 */
	snd_kcontrol_info_t *info;
	/* get回调函数用于读取控件的当前值 */
	snd_kcontrol_get_t *get;
	/* put回调函数用于把应用程序的控制值设置到控件中 */
	snd_kcontrol_put_t *put;
	union {
		snd_kcontrol_tlv_rw_t *c;
		const unsigned int *p;
	} tlv;
	/* 存储了struct soc_mixer_control的信息 */
	unsigned long private_value;
};
```

通常可以分成3部分来定义控件的名字：源——方向——功能，kernel文档中关于kcontrol命名[https://www.kernel.org/doc/html/v6.3/sound/designs/control-names.html](https://www.kernel.org/doc/html/v6.3/sound/designs/control-names.html)


##### kcontrol的辅助宏
`SOC_SINGLE`宏，这种控件只一个控制量，比如一个开关
```c
// include/sound/soc.h
/*
 * xname: 控件的名字
 * reg: 控件对应寄存器的地址
 * shift: 控件在寄存器中的偏移
 * max: 控件可设置的最大值
 * invert: 设定值是否逻辑取反
 * private_value: 
 */
#define SOC_SINGLE(xname, reg, shift, max, invert) \
{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
	.info = snd_soc_info_volsw, .get = snd_soc_get_volsw,\
	.put = snd_soc_put_volsw, \
	.private_value = SOC_SINGLE_VALUE(reg, shift, max, invert, 0) }

#define SOC_SINGLE_VALUE(xreg, xshift, xmax, xinvert, xautodisable) \
	SOC_DOUBLE_VALUE(xreg, xshift, xshift, xmax, xinvert, xautodisable)

#define SOC_DOUBLE_VALUE(xreg, shift_left, shift_right, xmax, xinvert, xautodisable) \
	((unsigned long)&(struct soc_mixer_control) \
	{.reg = xreg, .rreg = xreg, .shift = shift_left, \
	.rshift = shift_right, .max = xmax, .platform_max = xmax, \
	.invert = xinvert, .autodisable = xautodisable})

/* 使用soc_mixer_control来描述该控件对应寄存器控制的详细信息 */
struct soc_mixer_control {
	int min, max, platform_max;
	int reg, rreg;
	unsigned int shift, rshift;
	unsigned int sign_bit;
	unsigned int invert:1;
	unsigned int autodisable:1;
#ifdef CONFIG_SND_SOC_TOPOLOGY
	struct snd_soc_dobj dobj;
#endif
};
```

`SOC_SINGLE_TLV`宏，主要定义那些有增益控制的控件，例如音量控制器，EQ均衡器等
```c
#define SOC_SINGLE_TLV(xname, reg, shift, max, invert, tlv_array) \
{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
	.access = SNDRV_CTL_ELEM_ACCESS_TLV_READ |\
		 SNDRV_CTL_ELEM_ACCESS_READWRITE,\
	.tlv.p = (tlv_array), \
	.info = snd_soc_info_volsw, .get = snd_soc_get_volsw,\
	.put = snd_soc_put_volsw, \
	.private_value = SOC_SINGLE_VALUE(reg, shift, max, invert, 0) }
```

`SOC_DOUBLE`宏，可以在同一个寄存器中控制两个相似的变量，最常用的就是用于一些立体声的控件，我们需要同时对左右声道进行控制，因为多了一个声道，参数也就相应地多了一个shift偏移

##### kcontrol创建过程
codec驱动在在进行kcontrol的定义后，会对`struct snd_soc_component_driver`的`controls`和`num_controls`成员进行填充。在声卡的初始化过程中，会调用`soc_probe_link_components()`对dai_link上的所有component进行初始化设置，其中包括了kcontrol的创建`snd_soc_add_component_controls()`，其创建过程如下：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240409143156.png)

##### es8336 kcontrol定义示例
```c
// sound/soc/codec/es8336.c
static const struct snd_kcontrol_new es8336_snd_controls[] = {
	/* HP OUT VOLUME */
	SOC_DOUBLE_TLV("HP Playback Volume", ES8336_CPHP_ICAL_VOL_REG18,
		       4, 0, 4, 1, hpout_vol_tlv),
	/* HPMIXER VOLUME Control */
	SOC_DOUBLE_TLV("HPMixer Gain", ES8336_HPMIX_VOL_REG16,
		       0, 4, 7, 0, hpmixer_gain_tlv),

	/* DAC Digital controls */
	SOC_DOUBLE_R_TLV("DAC Playback Volume", ES8336_DAC_VOLL_REG33,
			 ES8336_DAC_VOLR_REG34, 0, 0xC0, 1, dac_vol_tlv),

	SOC_SINGLE("Enable DAC Soft Ramp", ES8336_DAC_SET1_REG30, 4, 1, 1),
	SOC_SINGLE("DAC Soft Ramp Rate", ES8336_DAC_SET1_REG30, 2, 4, 0),

	SOC_ENUM("Playback Polarity", dacpol),
	SOC_SINGLE("DAC Notch Filter", ES8336_DAC_SET2_REG31, 6, 1, 0),
	SOC_SINGLE("DAC Double Fs Mode", ES8336_DAC_SET2_REG31, 7, 1, 0),
	SOC_SINGLE("DAC Volume Control-LeR", ES8336_DAC_SET2_REG31, 2, 1, 0),
	SOC_SINGLE("DAC Stereo Enhancement", ES8336_DAC_SET3_REG32, 0, 7, 0),

	/* +20dB D2SE PGA Control */
	SOC_SINGLE_TLV("MIC Boost", ES8336_ADC_D2SEPGA_REG24,
		       0, 1, 0, mic_bst_tlv),
	/* 0-+24dB Lineinput PGA Control */
	SOC_SINGLE_TLV("Input PGA", ES8336_ADC_PGAGAIN_REG23,
		       4, 8, 0, linin_pga_tlv),
};

/* 填充到struct snd_soc_component_driver */
static const struct snd_soc_component_driver soc_component_dev_es8336 = {
	.probe =	es8336_probe,
	.remove =	es8336_remove,
	.suspend =	es8336_suspend,
	.resume =	es8336_resume,
	.set_bias_level = es8336_set_bias_level,

	.controls = es8336_snd_controls,
	.num_controls = ARRAY_SIZE(es8336_snd_controls),
	.dapm_widgets = es8336_dapm_widgets,
	.num_dapm_widgets = ARRAY_SIZE(es8336_dapm_widgets),
	.dapm_routes = es8336_dapm_routes,
	.num_dapm_routes = ARRAY_SIZE(es8336_dapm_routes),
};
```

##### 应用层访问Control设备
Control设备的文件操作集，`struct file_operations snd_ctl_f_ops`
```c
// sound/core/control.c
static const struct file_operations snd_ctl_f_ops =
{
	.owner =	THIS_MODULE,
	.read =		snd_ctl_read,
	.open =		snd_ctl_open,
	.release =	snd_ctl_release,
	.llseek =	no_llseek,
	.poll =		snd_ctl_poll,
	.unlocked_ioctl =	snd_ctl_ioctl,
	.compat_ioctl =	snd_ctl_ioctl_compat,
	.fasync =	snd_ctl_fasync,
};
```

### PCM设备
PCM设备是挂载`struct snd_card`的devices链表下的一个snd device，一个pcm实例由一个playback stream和一个capture stream组成，这两个stream又分别有一个或多个substream组成，在嵌入式系统中，通常不会有这么复杂，大多数情况下是一个声卡，一个pcm实例，pcm下面有一个playback stream和capture stream，playback和capture下面各自有一个substream

#### ALSA中pcm相关的数据结构
`struct snd_pcm`，pcm使用`struct snd_pcm`数据结构来描述，一个pcm实例由一个playback stream和一个capture stream组成，这两个stream又分别有一个或者多个substream组成
```c
// include/sound/pcm.h
struct snd_pcm {
	/* 指向pcm_runtime->card->snd_card */
	struct snd_card *card;
	struct list_head list;
	/* pcm设备编号 这里一般情况下就是rtd->num的值 */
	int device; /* device number */
	unsigned int info_flags;
	unsigned short dev_class;
	unsigned short dev_subclass;
	char id[64];
	char name[80];
	/* streams[0]表示placyback stream，streams[1]表示capture stream */
	struct snd_pcm_str streams[2];
	struct mutex open_mutex;
	wait_queue_head_t open_wait;
	void *private_data;
	void (*private_free) (struct snd_pcm *pcm);
	bool internal; /* pcm is for internal use only */
	bool nonatomic; /* whole PCM operations are in non-atomic context */
	bool no_device_suspend; /* don't invoke device PM suspend */
#if IS_ENABLED(CONFIG_SND_PCM_OSS)
	struct snd_pcm_oss oss;
#endif
};
```

`struct snd_pcm_str`，用于表示pcm stream，snd_pcm_str的主要作用就是指向`snd_pcm_substream`
```c
struct snd_pcm_str {
	/* 
	 * stream方向：
	 * SNDRV_PCM_STREAM_PLAYBACK表示播放
	 * SNDRV_PCM_STREAM_CAPTURE表示捕获 
	 */
	int stream;				/* stream (direction) */
	/* 指向所属pcm */
	struct snd_pcm *pcm;
	/* -- substreams -- */
	/* substream的个数 */
	unsigned int substream_count;
	/* substream的打开标志 */
	unsigned int substream_opened;
	/* 用来链接所有的substream */
	struct snd_pcm_substream *substream;
#if IS_ENABLED(CONFIG_SND_PCM_OSS)
	/* -- OSS things -- */
	struct snd_pcm_oss_stream oss;
#endif
#ifdef CONFIG_SND_VERBOSE_PROCFS
	struct snd_info_entry *proc_root;
#ifdef CONFIG_SND_PCM_XRUN_DEBUG
	unsigned int xrun_debug;	/* 0 = disabled, 1 = verbose, 2 = stacktrace */
#endif
#endif
	struct snd_kcontrol *chmap_kctl; /* channel-mapping controls */
	struct device dev;
};
```

`struct snd_pcm_substream`，用于表pcm substream
```c
struct snd_pcm_substream {
	/* 指向对应的snd_pcm和snd_pcm_str */
	struct snd_pcm *pcm;
	struct snd_pcm_str *pstr;
	/* 指向pcm_runtime */
	void *private_data;		/* copied from pcm->private_data */
	int number;
	char name[32];			/* substream name */
	int stream;			/* stream (direction) */
	struct pm_qos_request latency_pm_qos_req; /* pm_qos request */
	size_t buffer_bytes_max;	/* limit ring buffer size */
	struct snd_dma_buffer dma_buffer;
	size_t dma_max;
	/* -- hardware operations -- */
	const struct snd_pcm_ops *ops;
	/* -- runtime information -- */
	/* pcm 运行时实例，读写数据的时候由它来控制 */
	struct snd_pcm_runtime *runtime;
        /* -- timer section -- */
	struct snd_timer *timer;		/* timer */
	unsigned timer_running: 1;	/* time is running */
	long wait_time;	/* time in ms for R/W to wait for avail */
	/* -- next substream -- */
	struct snd_pcm_substream *next;
	/* -- linked substreams -- */
	struct list_head link_list;	/* linked list member */
	struct snd_pcm_group self_group;	/* fake group for non linked substream (with substream lock inside) */
	struct snd_pcm_group *group;		/* pointer to current group */
	/* -- assigned files -- */
	int ref_count;
	atomic_t mmap_count;
	unsigned int f_flags;
	void (*pcm_release)(struct snd_pcm_substream *);
	struct pid *pid;
#if IS_ENABLED(CONFIG_SND_PCM_OSS)
	/* -- OSS things -- */
	struct snd_pcm_oss_substream oss;
#endif
#ifdef CONFIG_SND_VERBOSE_PROCFS
	struct snd_info_entry *proc_root;
#endif /* CONFIG_SND_VERBOSE_PROCFS */
	/* misc flags */
	unsigned int hw_opened: 1;
	unsigned int managed_buffer_alloc:1;
};
```

`struct snd_pcm_hw_params`结构体，用于配置音频硬件参数的结构体，比如通道数、采样率、数据格式等
```c
// includ/uapi/sound/asound.h
struct snd_pcm_hw_params {
	unsigned int flags;
	struct snd_mask masks[SNDRV_PCM_HW_PARAM_LAST_MASK -
			       SNDRV_PCM_HW_PARAM_FIRST_MASK + 1];
	struct snd_mask mres[5];	/* reserved masks */
	/* 用于描述各个硬件参数的取值范围 */
	struct snd_interval intervals[SNDRV_PCM_HW_PARAM_LAST_INTERVAL -
				        SNDRV_PCM_HW_PARAM_FIRST_INTERVAL + 1];
	struct snd_interval ires[9];	/* reserved intervals */
	unsigned int rmask;		/* W: requested masks */
	unsigned int cmask;		/* R: changed masks */
	unsigned int info;		/* R: Info flags for returned setup */
	unsigned int msbits;		/* R: used most significant bits */
	unsigned int rate_num;		/* R: rate numerator */
	unsigned int rate_den;		/* R: rate denominator */
	snd_pcm_uframes_t fifo_size;	/* R: chip FIFO size in frames */
	unsigned char reserved[64];	/* reserved for future */
};
```

#### ASoC中pcm相关的数据结构

`struct snd_soc_pcm_stream`，用于描述SoC pcm stream的信息
```c
// include/sound/soc.h
/* SoC PCM stream information */
struct snd_soc_pcm_stream {
	const char *stream_name;
	/* 数据传输的位宽 */
	u64 formats;			/* SNDRV_PCM_FMTBIT_* */
	/* 音频采样率 */
	unsigned int rates;		/* SNDRV_PCM_RATE_* */
	/* 最低 最高采样率 */
	unsigned int rate_min;		/* min rate */
	unsigned int rate_max;		/* max rate */
	/* 最少通道数 最大通道数 */
	unsigned int channels_min;	/* min channels */
	unsigned int channels_max;	/* max channels */
	unsigned int sig_bits;		/* number of bits of content */
};
```

`struct snd_soc_pcm_runtime`，在注册声卡时会调用`soc_new_pcm_runtime()`为每一个`dai_link`分配一个`snd_soc_pcm_runtime`
```c
// include/sound/soc.h
/* SoC machine DAI configuration, glues a codec and cpu DAI together */
struct snd_soc_pcm_runtime {
	struct device *dev;
	/* 指向ASoC sound card */
	struct snd_soc_card *card;
	/* 指向ASoC sound card中对应的dai_link */
	struct snd_soc_dai_link *dai_link;
	struct snd_pcm_ops ops;

	unsigned int params_select; /* currently selected param for dai link */

	/* Dynamic PCM BE runtime data */
	struct snd_soc_dpcm_runtime dpcm[2];

	long pmdown_time;

	/* runtime devices */
	struct snd_pcm *pcm;
	struct snd_compr *compr;

	/*
	 * dais = cpu_dai + codec_dai
	 * see
	 *	soc_new_pcm_runtime()
	 *	asoc_rtd_to_cpu()
	 *	asoc_rtd_to_codec()
	 */
	/* 用于保存当前音频数据链路上的dai，包括cpu dai, codec dai */
	struct snd_soc_dai **dais;
	unsigned int num_codecs;
	unsigned int num_cpus;

	struct snd_soc_dapm_widget *playback_widget;
	struct snd_soc_dapm_widget *capture_widget;

	struct delayed_work delayed_work;
	void (*close_delayed_work_func)(struct snd_soc_pcm_runtime *rtd);
#ifdef CONFIG_DEBUG_FS
	struct dentry *debugfs_dpcm_root;
#endif

	/* 为ASoC声卡设备的每个pcm runtime会分配一个编号，从0开始 */
	unsigned int num; /* 0-based and monotonic increasing */
	/* 将当前的pcm runtime链接到ASoC sound card的rtd_list中 */
	struct list_head list; /* rtd list of the soc card */

	/* function mark */
	struct snd_pcm_substream *mark_startup;
	struct snd_pcm_substream *mark_hw_params;
	struct snd_pcm_substream *mark_trigger;
	struct snd_compr_stream  *mark_compr_startup;

	/* bit field */
	unsigned int pop_wait:1;
	unsigned int fe_compr:1; /* for Dynamic PCM */

	int num_components;
	/* 指向存储当前dai_link上的platform以及codec的component */
	struct snd_soc_component *components[]; /* CPU/Codec/Platform */
};
```

#### PCM设备创建过程

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240410153336.png)


### DAPM相关
DAPM是Dynamic Audio Power Management的缩写，即动态音频电源管理，是独立于内核其他PM的一套音频电源管理系统，DAPM对所有用户应用程序来说是完全透明的，电源切换的过程都在ASoC核心内完成，DAPM根据当前激活的音频流对声卡中的Mixer等进行配置，来决定音频控件的电源打开和关闭，达到省电的目的

#### Widget介绍
Widget是具备路径和电源管理的kcontrol，可以理解为kcontrol的进一步升级和封装，ASoC把系统划分为多个dapm域，每个widget属于某个dapm域，比如同一个codec中的widgets通常位于同一个dapm域，而platform上的widget可能又会位于另一个dapm域中

##### 相关数据结构
`struct snd_soc_dapm_widget`，用来描述Widget
```c
// include/sound/soc_dapm.h
/* dapm widget */
struct snd_soc_dapm_widget {
	enum snd_soc_dapm_type id;
	const char *name;		/* widget name */
	const char *sname;	/* stream name */
	/* 链接到snd_soc_card的widgets链表 */
	struct list_head list;
	struct snd_soc_dapm_context *dapm;

	void *priv;				/* widget specific data */
	struct regulator *regulator;		/* attached regulator */
	struct pinctrl *pinctrl;		/* attached pinctrl */

	/* dapm control */
	/* 用于dapm控制的寄存器地址 */
	int reg;				/* negative reg = no direct dapm */
	unsigned char shift;			/* bits to shift */
	unsigned int mask;			/* non-shifted mask */
	unsigned int on_val;			/* on state value */
	unsigned int off_val;			/* off state value */
	/* 表示widget的上电状态 */
	unsigned char power:1;			/* block power status */
	unsigned char active:1;			/* active stream on DAC, ADC's */
	unsigned char connected:1;		/* connected codec pin */
	unsigned char new:1;			/* cnew complete */
	unsigned char force:1;			/* force state */
	unsigned char ignore_suspend:1;         /* kept enabled over suspend */
	unsigned char new_power:1;		/* power from this run */
	unsigned char power_checked:1;		/* power checked this run */
	unsigned char is_supply:1;		/* Widget is a supply type widget */
	unsigned char is_ep:2;			/* Widget is a endpoint type widget */
	int subseq;				/* sort within widget type */

	int (*power_check)(struct snd_soc_dapm_widget *w);

	/* external events */
	unsigned short event_flags;		/* flags to specify event types */
	int (*event)(struct snd_soc_dapm_widget*, struct snd_kcontrol *, int);

	/* kcontrols that relate to this widget */
	int num_kcontrols;
	const struct snd_kcontrol_new *kcontrol_news;
	struct snd_kcontrol **kcontrols;
	struct snd_soc_dobj dobj;

	/* widget input and output edges */
	/* edge[0] 链接输出端连接的struct snd_soc_dapm_path */
	/* edge[1] 链接输入端连接的struct snd_soc_dapm_path */
	struct list_head edges[2];

	/* used during DAPM updates */
	struct list_head work_list;
	struct list_head power_list;
	struct list_head dirty;
	/*
	 * endpoints[0]保存从当前widget向前遍历，连接至SND_SOC_DAPM_EP_SOURCE
	 * 类型端点的路径数量
	 * endpoints[1]保存从当前widget向后遍历，连接至SND_SOC_DAPM_EP_SINK
	 * 类型端点的路径数量
	 */
	int endpoints[2];

	struct clk *clk;

	int channel;
};
```

`struct snd_soc_dapm_context`，描述dapm域
```c
// include/sound/soc_dapm.h
/* DAPM context */
struct snd_soc_dapm_context {
	enum snd_soc_bias_level bias_level;
	unsigned int idle_bias_off:1; /* Use BIAS_OFF instead of STANDBY */
	/* Go to BIAS_OFF in suspend if the DAPM context is idle */
	unsigned int suspend_bias_off:1;

	struct device *dev; /* from parent - for debug */
	/* 所属的component */
	struct snd_soc_component *component; /* parent component */
	/* 所属的snd_soc_card */
	struct snd_soc_card *card; /* parent card */

	/* used during DAPM updates */
	enum snd_soc_bias_level target_bias_level;
	/* 链接到snd_soc_card的dapm_list链表中 */
	struct list_head list;

	struct snd_soc_dapm_wcache path_sink_cache;
	struct snd_soc_dapm_wcache path_source_cache;

#ifdef CONFIG_DEBUG_FS
	struct dentry *debugfs_dapm;
#endif
};
```

##### 辅助宏定义widget
DAPM系统提供的一些辅助宏定义来定义各种类型的widget和它所用到的dapm kcontrol（这些由DAPM系统提供的辅助宏定义的kcontrol我们统一称为dapm kcontrol，以便和普通的kcontrol进行区分），这些宏定义在`include/sound/soc_dapm.h`中，分类为了4个不同域的widget辅助定义宏

platform域widget，一般是一些需要物理连接的输入、输出接口，这些widget是没有寄存器控制位来控制widget的电源状态的，因此reg字段被设置为SND_SOC_NOPM
```c
// include/sound/soc_dapm.h
/* platform domain */
#define SND_SOC_DAPM_SIGGEN(wname) \
{	.id = snd_soc_dapm_siggen, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM }
#define SND_SOC_DAPM_SINK(wname) \
{	.id = snd_soc_dapm_sink, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM }
/* 对应codec芯片的输入引脚 */
#define SND_SOC_DAPM_INPUT(wname) \
{	.id = snd_soc_dapm_input, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM }
/* 对应codec芯片的输出引脚 */
#define SND_SOC_DAPM_OUTPUT(wname) \
{	.id = snd_soc_dapm_output, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM }
/* 对应麦克风 */
#define SND_SOC_DAPM_MIC(wname, wevent) \
{	.id = snd_soc_dapm_mic, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM, .event = wevent, \
	.event_flags = SND_SOC_DAPM_PRE_PMU | SND_SOC_DAPM_POST_PMD}
/* 对应耳机 */
#define SND_SOC_DAPM_HP(wname, wevent) \
{	.id = snd_soc_dapm_hp, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM, .event = wevent, \
	.event_flags = SND_SOC_DAPM_POST_PMU | SND_SOC_DAPM_PRE_PMD}
#define SND_SOC_DAPM_SPK(wname, wevent) \
{	.id = snd_soc_dapm_spk, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM, .event = wevent, \
	.event_flags = SND_SOC_DAPM_POST_PMU | SND_SOC_DAPM_PRE_PMD}
#define SND_SOC_DAPM_LINE(wname, wevent) \
{	.id = snd_soc_dapm_line, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM, .event = wevent, \
	.event_flags = SND_SOC_DAPM_POST_PMU | SND_SOC_DAPM_PRE_PMD}

#define SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert) \
	.reg = wreg, .mask = 1, .shift = wshift, \
	.on_val = winvert ? 0 : 1, .off_val = winvert ? 1 : 0
```

path域widget，一般指codec内部的Mixer、Mux、Switch、Demux等控制音频路径的widget，这些widget是有相应的寄存器控制的，DAPM框架在扫描和更新音频路径时，会利用这些寄存器来控制widget的电源状态

es8336的示例，通过es8336的datasheet可知，Left Hp Mixer通过ES8336_HPMIX_PDN_REG15的bit4来控制Left Hp Mixer的电源状态，而es8336_out_left_mixer是通过ES8336_HPMIX_SWITCH_REG14寄存器的bit7来控制Left DAC的开关
```c
/* Output mixer  */
SND_SOC_DAPM_MIXER("Left Hp mixer", ES8336_HPMIX_PDN_REG15,
			4, 1, &es8336_out_left_mix[0],
			ARRAY_SIZE(es8336_out_left_mix)),

/* headphone Output Mixer */
static const struct snd_kcontrol_new es8336_out_left_mix[] = {
	SOC_DAPM_SINGLE("LLIN Switch", ES8336_HPMIX_SWITCH_REG14,
			6, 1, 0),
	SOC_DAPM_SINGLE("Left DAC Switch", ES8336_HPMIX_SWITCH_REG14,
			7, 1, 0),
};
```

stream域widget，指那些需要处理音频数据流的widget，例如ADC、DAC、AIF IN、AIF OUT等
```c
// include/sound/soc-dapm.h
/* stream domain */
#define SND_SOC_DAPM_AIF_IN(wname, stname, wchan, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_aif_in, .name = wname, .sname = stname, \
	.channel = wchan, SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
#define SND_SOC_DAPM_AIF_IN_E(wname, stname, wchan, wreg, wshift, winvert, \
			      wevent, wflags)				\
{	.id = snd_soc_dapm_aif_in, .name = wname, .sname = stname, \
	.channel = wchan, SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags }
#define SND_SOC_DAPM_AIF_OUT(wname, stname, wchan, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_aif_out, .name = wname, .sname = stname, \
	.channel = wchan, SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
#define SND_SOC_DAPM_AIF_OUT_E(wname, stname, wchan, wreg, wshift, winvert, \
			     wevent, wflags)				\
{	.id = snd_soc_dapm_aif_out, .name = wname, .sname = stname, \
	.channel = wchan, SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags }
#define SND_SOC_DAPM_DAC(wname, stname, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_dac, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert) }
#define SND_SOC_DAPM_DAC_E(wname, stname, wreg, wshift, winvert, \
			   wevent, wflags)				\
{	.id = snd_soc_dapm_dac, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags}

#define SND_SOC_DAPM_ADC(wname, stname, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_adc, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
#define SND_SOC_DAPM_ADC_E(wname, stname, wreg, wshift, winvert, \
			   wevent, wflags)				\
{	.id = snd_soc_dapm_adc, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags}
#define SND_SOC_DAPM_CLOCK_SUPPLY(wname) \
{	.id = snd_soc_dapm_clock_supply, .name = wname, \
	.reg = SND_SOC_NOPM, .event = dapm_clock_event, \
	.event_flags = SND_SOC_DAPM_PRE_PMU | SND_SOC_DAPM_POST_PMD }
```

##### 辅助宏定义dapm kcontrol
对于音频路径上的Mixer或Mux类型的widget，他们包含了若干个kcontrol，dapm利用这些kcontrol完成音频的控制

es8336驱动中的示例，用来控制Left DAC的开关
```c
/* headphone Output Mixer */
static const struct snd_kcontrol_new es8336_out_left_mix[] = {
	SOC_DAPM_SINGLE("LLIN Switch", ES8336_HPMIX_SWITCH_REG14,
			6, 1, 0),
	SOC_DAPM_SINGLE("Left DAC Switch", ES8336_HPMIX_SWITCH_REG14,
			7, 1, 0),
```

#### Path介绍
DAPM提出了path的概念，path将一个widget的输出端和另一个widget的输入端连接在一起

path使用`struct snd_soc_dapm_path`来描述，当widget之间发生连接关系时，snd_soc_dapm_path作为连接者，它的source字段会指向路径起始端widget，sink字段会指向路径目标端的widget
```c
// include/sound/soc_dapm.h
/* dapm audio path between two widgets */
struct snd_soc_dapm_path {
	const char *name;

	/*
	 * source (input) and sink (output) widgets
	 * The union is for convience, since it is a lot nicer to type
	 * p->source, rather than p->node[SND_SOC_DAPM_DIR_IN]
	 */
	union {
		struct {
			/* 输入端widget */
			struct snd_soc_dapm_widget *source;
			/* 输出端widget */
			struct snd_soc_dapm_widget *sink;
		};
		struct snd_soc_dapm_widget *node[2];
	};

	/* status */
	/*
	 * connect用来表示source widget和sink widget之间的连接状态 
	 * 如果在source widget和sink widget中间有kcontrol可以控制通断，
	 * 那么connect表示的就是kcontrol的通断状态，对于两个直连的widget，
	 * connect值始终为1。
	 */
	u32 connect:1;	/* source and sink widgets are connected */
	u32 walking:1;  /* path is in the process of being walked */
	u32 weak:1;	/* path ignored for power management */
	u32 is_supply:1;	/* At least one of the connected widgets is a supply */

	int (*connected)(struct snd_soc_dapm_widget *source,
			 struct snd_soc_dapm_widget *sink);

	struct list_head list_node[2];
	struct list_head list_kcontrol;
	/* 链接到声卡的paths链表中 */
	struct list_head list;
};
```

##### complete path
complete path: 一条完整的音频路径必须要有起点和终点，我们把这些起点和终点widget称之为端点widget


下面这些类型的widget可以称为端点widget

| 分类 | widget类型 |
| :-: | :- |
| codec的输入输出引脚 | snd_soc_dapm_input, SND_SOC_DAPM_EP_SOURCE类型端点 |
| | snd_soc_dapm_output, SND_SOC_DAPM_EP_SINK类型端点 |
| codec外接音频设备 | snd_soc_dapm_mic, SND_SOC_DAPM_EP_SOURCE类型端点 |
| | snd_soc_dapm_hp, SND_SOC_DAPM_EP_SINK类型端点 |
| | snd_soc_dapm_spk, SND_SOC_DAPM_EP_SINK类型端点 |
| | snd_soc_dapm_line|
| dai widget | snd_soc_dapm_dai_in |
| | snd_soc_dapm_dai_out |

DAPM要给一个widget上电的其中一个条件是：这个widget位于一条完整路径上，即向前、向后查找连接的widget均能找到一个端点widget，并且路径上的widget、path都是处于连接状态的


#### Route介绍
一个路径包括：起始端widget -> path的输入端 -> path的输出 -> 目标端widget，DAPM使用`struct snd_soc_dapm_route`结构来描述这样一个完整路径route

`struct snd_soc_dapm_route`结构体，用来描述一个路径
```c
/*
 * DAPM audio route definition.
 *
 * Defines an audio route originating at source via control and finishing
 * at sink.
 */
struct snd_soc_dapm_route {
	/* 指向目标端widget的名称 */
	const char *sink;
	/* 指向负责控制该连接所对应的kcontrol名称 */
	const char *control;
	/* 指向起始端widget的名称 */
	const char *source;

	/* Note: currently only supported for links where source is a supply */
	int (*connected)(struct snd_soc_dapm_widget *source,
			 struct snd_soc_dapm_widget *sink);

	struct snd_soc_dobj dobj;
};
```

这里直接使用名称来描述连接关系，所有定义好的route，最后都要注册到dapm系统中，dapm会根据这些名字找出相应的widget，并动态地生成所需要的snd_soc_dapm_path结构，正确地处理各个链表和指针的关系，实现两个widget之间的连接。


### machine驱动 以Phytium DP音频为例
这里通过飞腾dp音频驱动来看一下DP声卡注册的过程

`struct snd_soc_card`结构体定义

```c
// sound/soc/phytium/pmdk_dp.c
static struct snd_soc_card pmdk = {
	.name = "PMDK-I2S",
	.owner = THIS_MODULE,

	.dapm_widgets = pmdk_dp_dapm_widgets,
	.num_dapm_widgets = ARRAY_SIZE(pmdk_dp_dapm_widgets),
	.controls = pmdk_controls,
	.num_controls = ARRAY_SIZE(pmdk_controls),
	.dapm_routes = pmdk_dp_audio_map,
	.num_dapm_routes = ARRAY_SIZE(pmdk_dp_audio_map),
};
```
`struct snd_soc_dai_link`的定义，这里用到了`SND_SOC_DAILINK_DEFS`及`SND_SOC_DAILINK_REG`
```c
SND_SOC_DAILINK_DEFS(pmdk_dp0_dai,
	DAILINK_COMP_ARRAY(COMP_CPU("phytium-i2s-dp0")),
	DAILINK_COMP_ARRAY(COMP_CODEC("hdmi-audio-codec.1346918656", "i2s-hifi")),
	DAILINK_COMP_ARRAY(COMP_PLATFORM("snd-soc-dummy")));

SND_SOC_DAILINK_DEFS(pmdk_dp1_dai,
	DAILINK_COMP_ARRAY(COMP_CPU("phytium-i2s-dp1")),
	DAILINK_COMP_ARRAY(COMP_CODEC("hdmi-audio-codec.1346918657", "i2s-hifi")),
	DAILINK_COMP_ARRAY(COMP_PLATFORM("snd-soc-dummy")));

SND_SOC_DAILINK_DEFS(pmdk_dp2_dai,
	DAILINK_COMP_ARRAY(COMP_CPU("phytium-i2s-dp2")),
	DAILINK_COMP_ARRAY(COMP_CODEC("hdmi-audio-codec.1346918658", "i2s-hifi")),
	DAILINK_COMP_ARRAY(COMP_PLATFORM("snd-soc-dummy")));

static struct snd_soc_dai_link pmdk_dai_local[] = {
{
	.name = "Phytium dp0-audio",
	.stream_name = "Playback",
	.dai_fmt = SMDK_DAI_FMT,
	.init = pmdk_dp0_init,
	SND_SOC_DAILINK_REG(pmdk_dp0_dai),
},{
	.name = "Phytium dp1-audio",
	.stream_name = "Playback",
	.dai_fmt = SMDK_DAI_FMT,
	.init = pmdk_dp1_init,
	SND_SOC_DAILINK_REG(pmdk_dp1_dai),
},
{
	.name = "Phytium dp2-audio",
	.stream_name = "Playback",
	.dai_fmt = SMDK_DAI_FMT,
	.init = pmdk_dp2_init,
	SND_SOC_DAILINK_REG(pmdk_dp2_dai),
},
};
```

展开其中一个dai_link为
```c
static struct snc_soc_dai_link_component pmdk_dp0_dai_cpus[] = {
    .dai_name = "phytium_i2s_dp0",
};
static struct snc_soc_dai_link_component pmdk_dp0_dai_codecs[] = {
    .name = "hdmi-audio-codec.1346918656",
    .dai_name = "i2s_hifi",
};
static struct snc_soc_dai_link_component pmdk_dp0_dai_platforms[] = {
    .name = "snd_soc_dummy",
};

SND_SOC_DAILINK_REG(pmdk_dp0_dai);

SND_SOC_DAILINK_REGx(pmdk_dp0_dai,
                     SND_SOC_DAILINK_REG3,
                     SND_SOC_DAILINK_REG2,
                     SND_SOC_DAILINK_REG1)(pmdk_dp0_dai)

SND_SOC_DAILINK_REG1(pmdk_dp0_dai)

SND_SOC_DAILINK_REG3(pmdk_dp0_dai_cpus, pmdk_dp0_dai_codecs, pmdk_dp0_dai_platforms)

.cpu            = pdmk_dp0_dai_cpus,
.num_cpus       = ARRAY_SIZE(pdmk_dp0_dai_cpus),
.codecs         = pmdk_dp0_dai_codecs,
.num_codecs     = ARRAY_SIZE(pdmk_dp0_dai_codecs),
.platforms      = pmdk_dp0_dai_platforms
.num_platforms  = ARRAY_SIZE(pdmk_dp0_dai_platforms)


static struct snd_soc_dai_link pmdk_dai_local[] = {
    {
        .name = "Phyitum dp0-audio",
        .stream_name = "Playback",
        .dai_fmt = SMDK_DAI_FMT,
        .init = pmdk_dp0_init,
        .cpu            = pdmk_dp0_dai_cpus,
        .num_cpus       = ARRAY_SIZE(pdmk_dp0_dai_cpus),
        .codecs         = pmdk_dp0_dai_codecs,
        .num_codecs     = ARRAY_SIZE(pdmk_dp0_dai_codecs),
        .platforms      = pmdk_dp0_dai_platforms
        .num_platforms  = ARRAY_SIZE(pdmk_dp0_dai_platforms)
    };
}
```

### machine驱动 以asoc-simple-card为例
ASoC声卡的注册从machine驱动开始，machine驱动相关的主要数据结构如下：
+ `struct snd_soc_card`: ASoC中的核心数据结构，对ASoC声卡进行抽象
+ `struct snd_soc_dai_link`: 用来描述音频数据链路，在snd_soc_dai_link中，指定了platform, codec, codec_dai, cpu_dai的名字
+ `struct snd_soc_dai_link_components`: 保存components的名称和设备树节点，用于后续查找component

#### 数据结构
`struct snd_soc_card`结构体，对ASoC声卡的抽象，machine驱动需要对其实现
```c
// include/sound/soc.h
/* SoC card */
struct snd_soc_card {
	/* 声卡的名称，如果使用设备树，解析simple-audio-card,name得到 */
	const char *name;
	const char *long_name;
	const char *driver_name;
	const char *components;
#ifdef CONFIG_DMI
	char dmi_longname[80];
#endif /* CONFIG_DMI */
	char topology_shortname[32];

	struct device *dev;
	/* 指向ALSA声卡 */
	struct snd_card *snd_card;
	struct module *owner;

	struct mutex mutex;
	struct mutex dapm_mutex;

	/* Mutex for PCM operations */
	struct mutex pcm_mutex;
	enum snd_soc_pcm_subclass pcm_subclass;

	int (*probe)(struct snd_soc_card *card);
	int (*late_probe)(struct snd_soc_card *card);
	int (*remove)(struct snd_soc_card *card);

	/* the pre and post PM functions are used to do any PM work before and
	 * after the codec and DAI's do any PM work. */
	int (*suspend_pre)(struct snd_soc_card *card);
	int (*suspend_post)(struct snd_soc_card *card);
	int (*resume_pre)(struct snd_soc_card *card);
	int (*resume_post)(struct snd_soc_card *card);

	/* callbacks */
	int (*set_bias_level)(struct snd_soc_card *,
			      struct snd_soc_dapm_context *dapm,
			      enum snd_soc_bias_level level);
	int (*set_bias_level_post)(struct snd_soc_card *,
				   struct snd_soc_dapm_context *dapm,
				   enum snd_soc_bias_level level);

	int (*add_dai_link)(struct snd_soc_card *,
			    struct snd_soc_dai_link *link);
	void (*remove_dai_link)(struct snd_soc_card *,
			    struct snd_soc_dai_link *link);

	long pmdown_time;

	/* CPU <--> Codec DAI links  */
	/* 存储所有的dai link */
	struct snd_soc_dai_link *dai_link;  /* predefined links only */
	/* dai_link的数量 */
	int num_links;  /* predefined links only */
	/* 链接所有的snd_soc_pcm_runtime，一个dai_link对应一个runtime */
	struct list_head rtd_list;
	/* pcm runtime的数量 */
	int num_rtd;

	/* optional codec specific configuration */
	struct snd_soc_codec_conf *codec_conf;
	int num_configs;

	/*
	 * optional auxiliary devices such as amplifiers or codecs with DAI
	 * link unused
	 */
	struct snd_soc_aux_dev *aux_dev;
	int num_aux_devs;
	struct list_head aux_comp_list;

	const struct snd_kcontrol_new *controls;
	int num_controls;

	/*
	 * Card-specific routes and widgets.
	 * Note: of_dapm_xxx for Device Tree; Otherwise for driver build-in.
	 */
	const struct snd_soc_dapm_widget *dapm_widgets;
	int num_dapm_widgets;
	const struct snd_soc_dapm_route *dapm_routes;
	int num_dapm_routes;
	const struct snd_soc_dapm_widget *of_dapm_widgets;
	int num_of_dapm_widgets;
	const struct snd_soc_dapm_route *of_dapm_routes;
	int num_of_dapm_routes;

	/* lists of probed devices belonging to this card */
	struct list_head component_dev_list;
	struct list_head list;

	struct list_head widgets;
	struct list_head paths;
	struct list_head dapm_list;
	struct list_head dapm_dirty;

	/* attached dynamic objects */
	struct list_head dobj_list;

	/* Generic DAPM context for the card */
	struct snd_soc_dapm_context dapm;
	struct snd_soc_dapm_stats dapm_stats;
	struct snd_soc_dapm_update *update;

#ifdef CONFIG_DEBUG_FS
	struct dentry *debugfs_card_root;
#endif
#ifdef CONFIG_PM_SLEEP
	struct work_struct deferred_resume_work;
#endif
	u32 pop_time;

	/* bit field */
	unsigned int instantiated:1;
	unsigned int topology_shortname_created:1;
	unsigned int fully_routed:1;
	unsigned int disable_route_checks:1;
	unsigned int probed:1;
	unsigned int component_chaining:1;

	void *drvdata;
};
```

`struct snd_soc_dai_link`结构体，描述音频数据链路
```c
// include/sound/soc.h
struct snd_soc_dai_link {
	/* config - must be set by machine driver */
	const char *name;			/* Codec name */
	const char *stream_name;		/* Stream name */

	/*
	 * You MAY specify the link's CPU-side device, either by device name,
	 * or by DT/OF node, but not both. If this information is omitted,
	 * the CPU-side DAI is matched using .cpu_dai_name only, which hence
	 * must be globally unique. These fields are currently typically used
	 * only for codec to codec links, or systems using device tree.
	 */
	/*
	 * You MAY specify the DAI name of the CPU DAI. If this information is
	 * omitted, the CPU-side DAI is matched using .cpu_name/.cpu_of_node
	 * only, which only works well when that device exposes a single DAI.
	 */ 
	/* 保存当前dai_link上所有的cpu components */
	struct snd_soc_dai_link_component *cpus;
	/* cpu components的数量，一般来说1个dai_link只对应1个cpu和1个codec */
	unsigned int num_cpus;

	/*
	 * You MUST specify the link's codec, either by device name, or by
	 * DT/OF node, but not both.
	 */
	/* You MUST specify the DAI name within the codec */
	struct snd_soc_dai_link_component *codecs;
	unsigned int num_codecs;

	/*
	 * You MAY specify the link's platform/PCM/DMA driver, either by
	 * device name, or by DT/OF node, but not both. Some forms of link
	 * do not need a platform. In such case, platforms are not mandatory.
	 */
	struct snd_soc_dai_link_component *platforms;
	unsigned int num_platforms;

	int id;	/* optional ID for machine driver link identification */

	const struct snd_soc_pcm_stream *params;
	unsigned int num_params;

	/* 数字音频接口格式 */
	unsigned int dai_fmt;           /* format to set on init */

	enum snd_soc_dpcm_trigger trigger[2]; /* trigger type for DPCM */

	/* codec/machine specific init - e.g. add machine controls */
	int (*init)(struct snd_soc_pcm_runtime *rtd);

	/* codec/machine specific exit - dual of init() */
	void (*exit)(struct snd_soc_pcm_runtime *rtd);

	/* optional hw_params re-writing for BE and FE sync */
	int (*be_hw_params_fixup)(struct snd_soc_pcm_runtime *rtd,
			struct snd_pcm_hw_params *params);

	/* machine stream operations */
	const struct snd_soc_ops *ops;
	const struct snd_soc_compr_ops *compr_ops;

	/* Mark this pcm with non atomic ops */
	unsigned int nonatomic:1;

	/* For unidirectional dai links */
	/* 仅支持播放和录制标志 */
	unsigned int playback_only:1;
	unsigned int capture_only:1;

	/* Keep DAI active over suspend */
	/* 是否在挂起时保持DAI的活动状态 */
	unsigned int ignore_suspend:1;

	/* Symmetry requirements */
	unsigned int symmetric_rate:1;
	unsigned int symmetric_channels:1;
	unsigned int symmetric_sample_bits:1;

	/* Do not create a PCM for this DAI link (Backend link) */
	unsigned int no_pcm:1;

	/* This DAI link can route to other DAI links at runtime (Frontend)*/
	unsigned int dynamic:1;

	/* DPCM capture and Playback support */
	unsigned int dpcm_capture:1;
	unsigned int dpcm_playback:1;

	/* DPCM used FE & BE merged format */
	unsigned int dpcm_merged_format:1;
	/* DPCM used FE & BE merged channel */
	unsigned int dpcm_merged_chan:1;
	/* DPCM used FE & BE merged rate */
	unsigned int dpcm_merged_rate:1;

	/* pmdown_time is ignored at stop */
	unsigned int ignore_pmdown_time:1;

	/* Do not create a PCM for this DAI link (Backend link) */
	unsigned int ignore:1;

	/* This flag will reorder stop sequence. By enabling this flag
	 * DMA controller stop sequence will be invoked first followed by
	 * CPU DAI driver stop sequence
	 */
	unsigned int stop_dma_first:1;

#ifdef CONFIG_SND_SOC_TOPOLOGY
	struct snd_soc_dobj dobj; /* For topology */
#endif
};
```

`aso_simple_priv`结构体，asoc-simple-card驱动中集成的数据结构体
```c
// include/sound/simple_card_util.h
struct asoc_simple_priv {
	struct snd_soc_card snd_card;
	/* dai_props是和dai_link对应的 */
	struct simple_dai_props {
		struct asoc_simple_dai *cpu_dai;
		struct asoc_simple_dai *codec_dai;
		struct snd_soc_dai_link_component *cpus;
		struct snd_soc_dai_link_component *codecs;
		struct snd_soc_dai_link_component *platforms;
		struct asoc_simple_data adata;
		struct snd_soc_codec_conf *codec_conf;
		struct prop_nums num;
		unsigned int mclk_fs;
	} *dai_props;
	struct asoc_simple_jack hp_jack;
	struct asoc_simple_jack mic_jack;
	struct snd_soc_dai_link *dai_link;
	struct asoc_simple_dai *dais;
	struct snd_soc_dai_link_component *dlcs;
	struct snd_soc_dai_link_component dummy;
	struct snd_soc_codec_conf *codec_conf;
	struct gpio_desc *pa_gpio;
	/* ALSA PCM操作集 */
	const struct snd_soc_ops *ops;
	unsigned int dpcm_selectable:1;
	unsigned int force_dpcm:1;
};
```

`struct link_info`用于保存dai_link信息
```c
struct link_info {
	/* 表示音频数据链路的数量 */
	int link; /* number of link */
	/* 表示当前正在处理的是第几个cpu */
	int cpu;  /* turn for CPU / Codec */
	/* 保存每条音频数据链路所需的信息 */
	struct prop_nums num[SNDRV_MAX_LINKS];
};

struct prop_nums {
	/* 分别表示该dai_link下cpu codec platform的数量 */
	int cpus;
	int codecs;
	int platforms;
};
```

#### asoc-simple-card设备树节点描述
E2000Q设备树sound-card节点描述
```c
sound_card: sound {
	compatible = "simple-audio-card";
	simple-audio-card,format = "i2s";
	simple-audio-card,name = "phytium,pe220x-i2s-audio";
	simple-audio-card,pin-switches = "mic-in";
	simple-audio-card,widgets = "Microphone","mic-in";
	simple-audio-card,routing = "MIC2","mic-in";
	simple-audio-card,cpu {
		sound-dai = <&i2s0>;
	};
	simple-audio-card,codec{
		sound-dai = <&codec0>;
	};
};
```

simple-audio-card的各个属性设置[simple-card.yaml](https://www.kernel.org/doc/Documentation/devicetree/bindings/sound/simple-card.yaml)
1. `simple-audio-card,name`： 指定声卡的名称为“phytium,pe220x-i2s-audio”;
2. `simple-audio-card,audio`： 指定数字音频接口格式为`i2s`
3. `simple-audio-card,widge` ： 在alsa驱动中，使用widget描述具有路径电源管理的kcontrol，每个条目都是一对字符串，其中第一个是widget模板名称，在machine驱动中只有确定的几种"Microphone（表示麦克风）"、“Headphone（表示耳机）”、“Speaker（表示扬声器）”、“Line（表示线路）”；第二个是widget实例名称，可以自由定义
4. `simple-audio-card,routing`：配置与codec物理输入引脚、物理输出引脚连接的路径，每个条目都是一对字符串，第一个事目的(sink)，第二个是源(source)，E2000设备树中的描述指的是将其中名字为`mic-in`的widget连接到名字为`MIC2`的widget
5. `simple-audio-card,cpu`：指定cpu接入音频编码的dai
6. `simple-audio-card,codec`：指定codec接入cpu的dai


asoc-simple-card machine驱动probe的流程


总结来说，machine驱动的工作内容如下
+ 构造一个`struct snd_soc_dai_link`，将cpu和codec关联起来
+ 负责创建`struct snd_soc_card`即asoc-sound-card这个结构体，走`devm_snd_soc_register_card()`将其注册到asoc中


### plarform驱动 以phytium i2s驱动为例
以phytium phytium-i2s.c platform驱动为例进行分析

#### 相关结构体

`struct i2s_bus`
```c
// sound/soc/phytium/local.h
struct i2s_bus {
	struct i2sc_bus core;
	struct snd_card *card;
	struct pci_dev *pci;
	struct mutex prepare_mutex;
};
```

`struct i2s_phytium`用于驱动集中管理
```c
// sound/soc/phytium/local.h
struct i2s_phytium {
	struct azx chip;
	struct snd_pcm_substream *substream;
	struct device *dev;
	struct device *pdev;
	u32 paddr;
	/* i2s 寄存器基地址 */
	void __iomem *regs;
	/* DDMA 寄存器基地址 */
	void __iomem *regs_db;
	int irq_id;

	/* for pending irqs */
	struct work_struct irq_pending_work;

	/* sync probing */
	struct completion probe_wait;
	struct work_struct probe_work;

	/* extra flags */
	unsigned int pcie:1;
	unsigned int irq_pending_warned:1;
	unsigned int probe_continued:1;
	unsigned int i2s_dp:1;

	unsigned int i2s_reg_comp1;
	unsigned int i2s_reg_comp2;
	struct clk *clk;
	// capability用来标识当前是否有播放和录音的能力
	unsigned int capability;
	unsigned int quirks;
	u32 fifo_th;
	int active;
	u32 xfer_resolution;
	u32 ccr;
	u32 clk_base;

	struct i2s_clk_config_data config;

	  /*azx_dev*/
	struct i2s_stream core;
};
```

`struct azx`结构体
```c
struct azx {
	struct i2s_bus bus;
	struct snd_card *card;
	struct pci_dev *pci;
	int dev_index;

	/* playback stream数量，默认是1 */
	int playback_streams;
	/* playback索引偏移，默认为0 */
	int playback_index_offset;
	/* capture stream数量，默认是1 */
	int capture_streams;
	/* capture索引偏移，默认在playback索引之后 */
	int capture_index_offset;
	int num_streams;

	/* Register interaction */
	const struct i2s_controller_ops *ops;

	/* locks */
	struct mutex open_mutex;

	/* PCM */
	struct list_head pcm_list;

	/* flags */
	int bdl_pos_adj;
	unsigned int running:1;
	unsigned int region_requested:1;
	unsigned int disabled:1;
};
```

`struct azx_dev`结构体
```c
struct azx_dev {
	struct i2s_stream core;
	unsigned int irq_pending:1;
};
```

`struct i2sc_bus`结构体
```c
struct i2s_io_ops {
	int (*dma_alloc_pages)(struct i2sc_bus *bus, int type, size_t size,
			       struct snd_dma_buffer *buf);
	void (*dma_free_pages)(struct i2sc_bus *bus,
			       struct snd_dma_buffer *buf);
};

// sound/soc/phytium/local.h
struct i2sc_bus {
	struct device *dev;
	const struct i2s_bus_ops *ops;
	/* 用于dma内存分配和释放的操作 */
	const struct i2s_io_ops *io_ops;
	const struct i2s_ext_bus_ops *ext_ops;

	/* h/w resources */
	/* I2S控制器寄存器基地址 */
	unsigned long addr;
	/* DMA控制器映射后的寄存器基地址 */
	void __iomem *remap_addr;
	/* 中断号 */
	int irq;

	/* codec linked list */
	struct list_head codec_list;
	unsigned int num_codecs;

	unsigned int unsol_rp, unsol_wp;
	struct work_struct unsol_work;

	struct snd_dma_buffer bdl0;
	struct snd_dma_buffer bdl1;

	/* i2s_stream linked list */
	struct list_head stream_list;

	bool reverse_assign;		/* assign devices in reverse order */

	int bdl_pos_adj;		/* BDL position adjustment */

	/* locks */
	spinlock_t reg_lock;
};
```

`struct i2s_stream`结构体
```c
struct i2s_stream {
	struct i2sc_bus *bus;
	/* 用于描述该stream的dma内存 */
	struct snd_dma_buffer bdl; /* BDL buffer */
	__le32 *posbuf;		/* position buffer pointer */
	/* stream的方向 */
	int direction;		/* playback / capture (SNDRV_PCM_STREAM_*) */

	unsigned int bufsize;	/* size of the play buffer in bytes */
	unsigned int period_bytes; /* size of the period in bytes */
	unsigned int frags;	/* number for period in the play buffer */
	unsigned int fifo_size;	/* FIFO size */

	/* DMA控制器映射后的寄存器基地址 */
	void __iomem *sd_addr;	/* stream descriptor pointer */

	u32 sd_int_sta_mask;	/* stream int status mask */

	/* pcm support */
	struct snd_pcm_substream *substream;	/* assigned substream,
						 * set in PCM open
						 */
	unsigned int format_val;	/* format value to be set in the
					 * controller and the codec
					 */
	/* stream的标记，意义应该不大 */
	unsigned char stream_tag;	/* assigned stream */
	/* stream的索引号 */
	unsigned char index;		/* stream index */
	int assigned_key;		/* last device# key assigned to */

	bool opened;
	bool running;
	bool prepared;
	bool no_period_wakeup;

	int delay_negative_threshold;

	struct list_head list;
};
```

#### platform driver注册过程
phytium i2s的设备树描述
```c
i2s0: i2s@28009000 {
	compatible = "phytium,i2s";
	reg = <0x0 0x28009000 0x0 0x1000>,
			<0x0 0x28005000 0x0 0x1000>;
	interrupts = <GIC_SPI 77 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&sysclk_600mhz>;
	clock-names = "i2s_clk";
	status = "disabled";
};
```



`struct snd_soc_component_driver phyitum_i2s_component`的定义
```c
// sound/soc/phytium/phytium_i2s.c
static const struct snd_soc_component_driver phytium_i2s_component = {
	.name			= "phytium-i2s",
	.pcm_construct	= phytium_pcm_new,
	.suspend		= phyitum_i2s_suspend,
	.resume 		= phytium_i2s_resume,

	.open			= phytium_pcm_open,
	.close			= phytium_pcm_close,
	.hw_params 		= phytium_pcm_hw_params,
	.prepare 		= phytium_pcm_prepare,
	.hw_free		= phytium_pcm_hw_free,
	.trigger		= phytium_pcm_trigger,
	.pointer 		= phytium_pcm_pointer,
};
```

`struct snd_soc_dai_driver phytium_i2s_dai`的定义
```c
static const struct snd_soc_dai_ops phytium_i2s_dai_ops = {
	.hw_params	= phytium_i2s_hw_params,
	.prepare	= phytium_i2s_prepare,
	.trigger	= phytium_i2s_trigger,
	.set_fmt	= phytium_i2s_set_fmt,
};

static struct snd_soc_dai_driver phytium_i2s_dai = {
	.playback = {
		.stream_name = "i2s-Playback",
		.channels_min = 2,
		.channels_max = 2,
		.rates = SNDRV_PCM_RATE_8000_192000,
		.formats = SNDRV_PCM_FMTBIT_S8 |
			   SNDRV_PCM_FMTBIT_S16_LE |
			   SNDRV_PCM_FMTBIT_S20_LE |
			   SNDRV_PCM_FMTBIT_S24_LE |
			   SNDRV_PCM_FMTBIT_S32_LE,
	},
	.capture = {
		.stream_name = "i2s-Capture",
		.channels_min = 2,
		.channels_max = 2,
		.rates = SNDRV_PCM_RATE_8000_192000,
		.formats = SNDRV_PCM_FMTBIT_S8 |
			   SNDRV_PCM_FMTBIT_S16_LE |
			   SNDRV_PCM_FMTBIT_S20_LE |
			   SNDRV_PCM_FMTBIT_S24_LE |
			   SNDRV_PCM_FMTBIT_S32_LE,
	},
	.ops     = &phytium_i2s_dai_ops,
	.symmetric_rate = 1,
};
```

### codec驱动 以es8336为例
codec driver不应包含任何特定于目标平台或者设备的代码，

描述codec的最主要的几个数据结构分别是：
+ `struct snd_soc_dai`: 描述codec端的dai
+ `struct snd_soc_dai_driver`: 描述dai的驱动，用于定义pcm的能力和操作集
+ `struct snd_soc_component`: ASoC使用统一的数据结构来描述codec设备和platform设备，一个component对应一个模块
+ `struct snd_soc_component_driver`: 描述component的驱动

#### 数据结构
`struct snd_soc_component`
```c
struct snd_soc_component {
	/* 
	 * component名称 dai_link可以通过这个字段在全局链表component_list中查找对应的
	 * component
	 */
	const char *name;
	int id;
	const char *name_prefix;
	struct device *dev;
	/* 该component绑定的snd_soc_card声卡 */
	struct snd_soc_card *card;

	unsigned int active;

	unsigned int suspended:1; /* is in suspend PM state */

	/* 链接到全局链表 component_list上 */
	struct list_head list;
	struct list_head card_aux_list; /* for auxiliary bound components */
	struct list_head card_list;

	const struct snd_soc_component_driver *driver;

	/* 用于管理component上的dai */
	struct list_head dai_list;
	/* component中dai的数量 */
	int num_dai;

	struct regmap *regmap;
	int val_bytes;

	struct mutex io_mutex;

	/* attached dynamic objects */
	struct list_head dobj_list;

	/*
	 * DO NOT use any of the fields below in drivers, they are temporary and
	 * are going to be removed again soon. If you use them in driver code
	 * the driver will be marked as BROKEN when these fields are removed.
	 */

	/* Don't use these, use snd_soc_component_get_dapm() */
	struct snd_soc_dapm_context dapm;

	/* machine specific init */
	int (*init)(struct snd_soc_component *component);

	/* function mark */
	struct snd_pcm_substream *mark_module;
	struct snd_pcm_substream *mark_open;
	struct snd_pcm_substream *mark_hw_params;
	struct snd_pcm_substream *mark_trigger;
	struct snd_compr_stream  *mark_compr_open;
	void *mark_pm;

#ifdef CONFIG_DEBUG_FS
	struct dentry *debugfs_root;
	const char *debugfs_prefix;
#endif
};
```

`struct snd_soc_component_driver`，用于component的driver，例如phytium i2s的platform驱动中定义的`phytium_i2s_component`实例
```c
// include/sound/soc-component.h
struct snd_soc_component_driver {
	const char *name;

	/* Default control and setup, added after probe() is run */
	/* 在codec的driver中一般会在驱动中直接进行定义并赋值到这几个成员 */
	const struct snd_kcontrol_new *controls;
	unsigned int num_controls;
	const struct snd_soc_dapm_widget *dapm_widgets;
	unsigned int num_dapm_widgets;
	const struct snd_soc_dapm_route *dapm_routes;
	unsigned int num_dapm_routes;

	/* 注册声卡时回调的probe函数 */
	int (*probe)(struct snd_soc_component *component);
	void (*remove)(struct snd_soc_component *component);
	int (*suspend)(struct snd_soc_component *component);
	int (*resume)(struct snd_soc_component *component);

	/* 用于读写寄存器的值，假如使用了regmap就不需要实现这两个接口了 */
	unsigned int (*read)(struct snd_soc_component *component,
			     unsigned int reg);
	int (*write)(struct snd_soc_component *component,
		     unsigned int reg, unsigned int val);

	/* pcm creation and destruction */
	int (*pcm_construct)(struct snd_soc_component *component,
			     struct snd_soc_pcm_runtime *rtd);
	void (*pcm_destruct)(struct snd_soc_component *component,
			     struct snd_pcm *pcm);

	/* component wide operations */
	int (*set_sysclk)(struct snd_soc_component *component,
			  int clk_id, int source, unsigned int freq, int dir);
	int (*set_pll)(struct snd_soc_component *component, int pll_id,
		       int source, unsigned int freq_in, unsigned int freq_out);
	int (*set_jack)(struct snd_soc_component *component,
			struct snd_soc_jack *jack,  void *data);

	/* DT */
	int (*of_xlate_dai_name)(struct snd_soc_component *component,
				 const struct of_phandle_args *args,
				 const char **dai_name);
	int (*of_xlate_dai_id)(struct snd_soc_component *comment,
			       struct device_node *endpoint);
	void (*seq_notifier)(struct snd_soc_component *component,
			     enum snd_soc_dapm_type type, int subseq);
	int (*stream_event)(struct snd_soc_component *component, int event);
	int (*set_bias_level)(struct snd_soc_component *component,
			      enum snd_soc_bias_level level);

	int (*open)(struct snd_soc_component *component,
		    struct snd_pcm_substream *substream);
	int (*close)(struct snd_soc_component *component,
		     struct snd_pcm_substream *substream);
	int (*ioctl)(struct snd_soc_component *component,
		     struct snd_pcm_substream *substream,
		     unsigned int cmd, void *arg);
	int (*hw_params)(struct snd_soc_component *component,
			 struct snd_pcm_substream *substream,
			 struct snd_pcm_hw_params *params);
	int (*hw_free)(struct snd_soc_component *component,
		       struct snd_pcm_substream *substream);
	int (*prepare)(struct snd_soc_component *component,
		       struct snd_pcm_substream *substream);
	int (*trigger)(struct snd_soc_component *component,
		       struct snd_pcm_substream *substream, int cmd);
	int (*sync_stop)(struct snd_soc_component *component,
			 struct snd_pcm_substream *substream);
	snd_pcm_uframes_t (*pointer)(struct snd_soc_component *component,
				     struct snd_pcm_substream *substream);
	int (*get_time_info)(struct snd_soc_component *component,
		struct snd_pcm_substream *substream, struct timespec64 *system_ts,
		struct timespec64 *audio_ts,
		struct snd_pcm_audio_tstamp_config *audio_tstamp_config,
		struct snd_pcm_audio_tstamp_report *audio_tstamp_report);
	int (*copy_user)(struct snd_soc_component *component,
			 struct snd_pcm_substream *substream, int channel,
			 unsigned long pos, void __user *buf,
			 unsigned long bytes);
	struct page *(*page)(struct snd_soc_component *component,
			     struct snd_pcm_substream *substream,
			     unsigned long offset);
	int (*mmap)(struct snd_soc_component *component,
		    struct snd_pcm_substream *substream,
		    struct vm_area_struct *vma);
	int (*ack)(struct snd_soc_component *component,
		   struct snd_pcm_substream *substream);

	const struct snd_compress_ops *compress_ops;

	/* probe ordering - for components with runtime dependencies */
	/* probe的顺序，一共有5个顺序，驱动初始化的过程中根据顺序进行component的probe */
	int probe_order;
	int remove_order;

	/*
	 * signal if the module handling the component should not be removed
	 * if a pcm is open. Setting this would prevent the module
	 * refcount being incremented in probe() but allow it be incremented
	 * when a pcm is opened and decremented when it is closed.
	 */
	unsigned int module_get_upon_open:1;

	/* bits */
	unsigned int idle_bias_on:1;
	unsigned int suspend_bias_off:1;
	unsigned int use_pmdown_time:1; /* care pmdown_time at stop */
	unsigned int endianness:1;
	unsigned int non_legacy_dai_naming:1;

	/* this component uses topology and ignore machine driver FEs */
	const char *ignore_machine;
	const char *topology_name_prefix;
	int (*be_hw_params_fixup)(struct snd_soc_pcm_runtime *rtd,
				  struct snd_pcm_hw_params *params);
	bool use_dai_pcm_id;	/* use DAI link PCM ID as PCM device number */
	int be_pcm_base;	/* base device ID for all BE PCMs */
};
```

`struct snd_soc_dai`，用来描述dai
```c
// include/sound/soc_dai.h
/*
 * Digital Audio Interface runtime data.
 *
 * Holds runtime data for a DAI.
 */
struct snd_soc_dai {
	const char *name;
	int id;
	struct device *dev;

	/* 指向dai_driver */
	/* driver ops */
	struct snd_soc_dai_driver *driver;

	/* DAI runtime info */
	unsigned int stream_active[SNDRV_PCM_STREAM_LAST + 1]; /* usage count */

	struct snd_soc_dapm_widget *playback_widget;
	struct snd_soc_dapm_widget *capture_widget;

	/* DAI DMA data */
	void *playback_dma_data;
	void *capture_dma_data;

	/* Symmetry data - only valid if symmetry is being enforced */
	unsigned int rate;
	unsigned int channels;
	unsigned int sample_bits;

	/* 指向component */
	/* parent platform/codec */
	struct snd_soc_component *component;

	/* CODEC TDM slot masks and params (for fixup) */
	unsigned int tx_mask;
	unsigned int rx_mask;

	/* 链接到component的dai_list链表 */
	struct list_head list;

	/* function mark */
	struct snd_pcm_substream *mark_startup;
	struct snd_pcm_substream *mark_hw_params;
	struct snd_pcm_substream *mark_trigger;
	struct snd_compr_stream  *mark_compr_startup;

	/* bit field */
	unsigned int probed:1;
};
```

`struct snd_soc_dai_driver`，描述dai驱动
```c
// include/sound/soc_dai.h
/*
 * Digital Audio Interface Driver.
 *
 * Describes the Digital Audio Interface in terms of its ALSA, DAI and AC97
 * operations and capabilities. Codec and platform drivers will register this
 * structure for every DAI they have.
 *
 * This structure covers the clocking, formating and ALSA operations for each
 * interface.
 */
struct snd_soc_dai_driver {
	/* DAI description */
	const char *name;
	/* dai标识符 */
	unsigned int id;
	unsigned int base;
	struct snd_soc_dobj dobj;

	/* DAI driver callbacks */
	int (*probe)(struct snd_soc_dai *dai);
	int (*remove)(struct snd_soc_dai *dai);
	/* compress dai */
	int (*compress_new)(struct snd_soc_pcm_runtime *rtd, int num);
	/* Optional Callback used at pcm creation*/
	int (*pcm_new)(struct snd_soc_pcm_runtime *rtd,
		       struct snd_soc_dai *dai);

	/* ops */
	/* dai的操作集，codec和platform驱动需要实现它 */
	const struct snd_soc_dai_ops *ops;
	const struct snd_soc_cdai_ops *cops;

	/* DAI capabilities */
	struct snd_soc_pcm_stream capture;
	struct snd_soc_pcm_stream playback;
	unsigned int symmetric_rate:1;
	unsigned int symmetric_channels:1;
	unsigned int symmetric_sample_bits:1;

	/* probe ordering - for components with runtime dependencies */
	int probe_order;
	int remove_order;
};
```

`struct snd_soc_dai_ops`，dai的控制和参数配置操作集结构体
```c
// include/sound/soc_dai.h
struct snd_soc_dai_ops {
	/*
	 * DAI clocking configuration, all optional.
	 * Called by soc_card drivers, normally in their hw_params.
	 */
	/* 配置dai的时钟 */
	int (*set_sysclk)(struct snd_soc_dai *dai,
		int clk_id, unsigned int freq, int dir);
	int (*set_pll)(struct snd_soc_dai *dai, int pll_id, int source,
		unsigned int freq_in, unsigned int freq_out);
	int (*set_clkdiv)(struct snd_soc_dai *dai, int div_id, int div);
	int (*set_bclk_ratio)(struct snd_soc_dai *dai, unsigned int ratio);

	/*
	 * DAI format configuration
	 * Called by soc_card drivers, normally in their hw_params.
	 */
	int (*set_fmt)(struct snd_soc_dai *dai, unsigned int fmt);
	int (*xlate_tdm_slot_mask)(unsigned int slots,
		unsigned int *tx_mask, unsigned int *rx_mask);
	int (*set_tdm_slot)(struct snd_soc_dai *dai,
		unsigned int tx_mask, unsigned int rx_mask,
		int slots, int slot_width);
	int (*set_channel_map)(struct snd_soc_dai *dai,
		unsigned int tx_num, unsigned int *tx_slot,
		unsigned int rx_num, unsigned int *rx_slot);
	int (*get_channel_map)(struct snd_soc_dai *dai,
			unsigned int *tx_num, unsigned int *tx_slot,
			unsigned int *rx_num, unsigned int *rx_slot);
	int (*set_tristate)(struct snd_soc_dai *dai, int tristate);

	int (*set_stream)(struct snd_soc_dai *dai,
			  void *stream, int direction);
	void *(*get_stream)(struct snd_soc_dai *dai, int direction);

	/*
	 * DAI digital mute - optional.
	 * Called by soc-core to minimise any pops.
	 */
	int (*mute_stream)(struct snd_soc_dai *dai, int mute, int stream);

	/*
	 * ALSA PCM audio operations - all optional.
	 * Called by soc-core during audio PCM operations.
	 */
	int (*startup)(struct snd_pcm_substream *,
		struct snd_soc_dai *);
	void (*shutdown)(struct snd_pcm_substream *,
		struct snd_soc_dai *);
	/* 配置PCM音频的硬件参数 */
	int (*hw_params)(struct snd_pcm_substream *,
		struct snd_pcm_hw_params *, struct snd_soc_dai *);
	/* 释放PCM音频的硬件资源 */
	int (*hw_free)(struct snd_pcm_substream *,
		struct snd_soc_dai *);
	/* 准备PCM音频操作 */
	int (*prepare)(struct snd_pcm_substream *,
		struct snd_soc_dai *);
	/*
	 * NOTE: Commands passed to the trigger function are not necessarily
	 * compatible with the current state of the dai. For example this
	 * sequence of commands is possible: START STOP STOP.
	 * So do not unconditionally use refcounting functions in the trigger
	 * function, e.g. clk_enable/disable.
	 */
	 /* 触发PCM音频操作 */
	int (*trigger)(struct snd_pcm_substream *, int,
		struct snd_soc_dai *);
	int (*bespoke_trigger)(struct snd_pcm_substream *, int,
		struct snd_soc_dai *);
	/*
	 * For hardware based FIFO caused delay reporting.
	 * Optional.
	 */
	snd_pcm_sframes_t (*delay)(struct snd_pcm_substream *,
		struct snd_soc_dai *);

	/*
	 * Format list for auto selection.
	 * Format will be increased if priority format was
	 * not selected.
	 * see
	 *	snd_soc_dai_get_fmt()
	 */
	u64 *auto_selectable_formats;
	int num_auto_selectable_formats;

	/* bit field */
	unsigned int no_capture_mute:1;
};
```


#### component注册过程
一般在codec和platform驱动中会调用`devm_snd_soc_register_component()`来注册component和dai，`devm_snd_soc_register_component`是带有资源管理的component注册函数，该函数会动态申请一个component，并将其添加到全局链表component_list中，同时会为每个dai driver动态分配一个dai，并建立dai与component、dai与dai driver的关系。

在Machine驱动中匹配codec，实际上就是根据音频数据链路snd_soc_dai_link codec指定的name去遍历component_list找到匹配的component，然后再根据dai_name从component->dai_list中获取到匹配的codec dai。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240410171031.png)



