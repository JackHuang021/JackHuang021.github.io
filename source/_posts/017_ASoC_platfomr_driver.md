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
+ 音频设备文件结构如下
![](https://raw.githubusercontent.com/JackHuang021/images/master/20221108112754.png)
    + controlC0: 用于card0声卡的控制
    + pcmC0D0c: 用于card0 device0录音的pcm设备
    + pcmC0D0p: 用于card0 device0播放的pcm设备
    + timer: 定时器
+ 一个声卡可以有多个设备，一个设备可以有多个播放\录音通道，每个设备节点都对应一个fops

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240124105117.png)


### ASoC驱动框架
+ ASoC（ALSA system on chip）是建立在标准ALSA驱动层上，对底层的ALSA框架封装了一层，为了更好的支持嵌入式CPU和音频编解码设备的一套软件体系，ASoC驱动主要由platform驱动、codec驱动、machine驱动组成。

machine驱动：充当描述和绑定其他组件驱动程序以形成ALSA声卡的粘合剂，machine可以理解为对声卡的抽象，它把cpu_dai，codec_dai通过dai_link链接起来，然后注册snd_soc_card。该驱动实现`struct snd_soc_card`的定义和注册，并通过指定`struct snd_soc_dai_link`中的`codec_name`, `platform_name`, `codec_dai_name`, `platform_dai_name`从而实现与其他各个驱动组件的绑定

platform驱动：包括音频dmaengine驱动咸亨粗，数字音频接口（DAI）驱动程序（例如I2S AC97 PCM），以及该平台的任何音频DSP驱动程序，向ALSA `platform_list`注册`snd_soc_platform_driver`，并通过`snd_soc_ops`暴露其音频能力

codec驱动：codec驱动程序独立于平台，包含音频控件，音频接口功能，编解码器DAPM定义和编解码器IO功能，如果需要，该类可以扩展至BT、FM和MODEM IC，codec类驱动应该是可以在任何体系结构和机器上运行的通用代码，通过注册`devm_snd_soc_register_component()`挂载自己的音频能力，通常codec驱动还需要注册kcontrol, widget, route等信息

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240124105147.png)

### 相关名词描述

DAPM（Dynamic Audio Power Management）：动态音频电源管理，用于Linux设备使用音频子系统内的最低电量，它独立于其他内核PM

DAI(Digital Audio Interface)：

Components：

Kcontrols：代表声卡里面的各种硬件开关，滑动控件等，通过软件定义kcontrol可以通过用户态配置硬件寄存器的开关，ALSA用snd_kcontrol_new定义kcontrol

widgets：DAPM通路切换是通过配置不同的route表来实现的，影响音频route开关的组件称为widget
影响path通路最常用的几种widget，如mixer, mux, switch等

mixer：多个输入混合成一个输出，可以使用SND_SOC_DAPM_MIXER来定义这种类型的widget，mixer的widget类型为snd_soc_dapm_mixer

mux：多路选择器，多路输入，但是只能选择一路作为输出，可以使用SND_SOC_DAPM_MUX来定义这个widget，widget类型为snd_soc_dapm_mux

pga：单路输入，单路输出，带gain调整的部件，可以使用SND_SOC_DAPM_PGA来定义这个widget，所属widget类型为snd_soc_dapm_pga

router：定义了mixer mux以及其它widget后需要定义route结构将它们连接起来，否则ALSA并不知道这些widget是如何连接的



### PCM数据流


### 相关结构体
`struct snd_card`
```c
// include/sound/core.h
/* main structure for soundcard */

struct snd_card {
	int number;			/* number of soundcard (index to
								snd_cards) */

	char id[16];			/* id string of this card */
	char driver[16];		/* driver name */
	char shortname[32];		/* short name of this soundcard */
	char longname[80];		/* name of this soundcard */
	char irq_descr[32];		/* Interrupt description */
	char mixername[80];		/* mixer name */
	char components[128];		/* card components delimited with
								space */
	struct module *module;		/* top-level module */

	void *private_data;		/* private data for soundcard */
	void (*private_free) (struct snd_card *card); /* callback for freeing of
								private data */
	struct list_head devices;	/* devices */

	struct device ctl_dev;		/* control device */
	unsigned int last_numid;	/* last used numeric ID */
	struct rw_semaphore controls_rwsem;	/* controls list lock */
	rwlock_t ctl_files_rwlock;	/* ctl_files list lock */
	int controls_count;		/* count of all controls */
	size_t user_ctl_alloc_size;	// current memory allocation by user controls.
	struct list_head controls;	/* all controls for this card */
	struct list_head ctl_files;	/* active control files */

	struct snd_info_entry *proc_root;	/* root for soundcard specific files */
	struct proc_dir_entry *proc_root_link;	/* number link to real id */

	struct list_head files_list;	/* all files associated to this card */
	struct snd_shutdown_f_ops *s_f_ops; /* file operations in the shutdown
								state */
	spinlock_t files_lock;		/* lock the files for this card */
	int shutdown;			/* this card is going down */
	struct completion *release_completion;
	struct device *dev;		/* device assigned to this card */
	struct device card_dev;		/* cardX object for sysfs */
	const struct attribute_group *dev_groups[4]; /* assigned sysfs attr */
	bool registered;		/* card_dev is registered? */
	bool managed;			/* managed via devres */
	bool releasing;			/* during card free process */
	int sync_irq;			/* assigned irq, used for PCM sync */
	wait_queue_head_t remove_sleep;

	size_t total_pcm_alloc_bytes;	/* total amount of allocated buffers */
	struct mutex memory_mutex;	/* protection for the above */
#ifdef CONFIG_SND_DEBUG
	struct dentry *debugfs_root;    /* debugfs root for card */
#endif

#ifdef CONFIG_PM
	unsigned int power_state;	/* power state */
	atomic_t power_ref;
	wait_queue_head_t power_sleep;
	wait_queue_head_t power_ref_sleep;
#endif

#if IS_ENABLED(CONFIG_SND_MIXER_OSS)
	struct snd_mixer_oss *mixer_oss;
	int mixer_oss_change_count;
#endif
};
```

`struct snd_soc_card`结构体，描述ASOC声卡，machine驱动需要对其实现
```c
// include/sound/soc.h
/* SoC card */
struct snd_soc_card {
	const char *name;
	const char *long_name;
	const char *driver_name;
	const char *components;
#ifdef CONFIG_DMI
	char dmi_longname[80];
#endif /* CONFIG_DMI */
	char topology_shortname[32];

	struct device *dev;
	/* 指向struct snd_card声卡 */
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

`struct snd_soc_component_driver`，用于component的driver，例如phytium i2s的platform驱动中定义的phytium_i2s_component实例
```c
// include/sound/soc-component.h
struct snd_soc_component_driver {
	const char *name;

	/* Default control and setup, added after probe() is run */
	const struct snd_kcontrol_new *controls;
	unsigned int num_controls;
	const struct snd_soc_dapm_widget *dapm_widgets;
	unsigned int num_dapm_widgets;
	const struct snd_soc_dapm_route *dapm_routes;
	unsigned int num_dapm_routes;

	int (*probe)(struct snd_soc_component *component);
	void (*remove)(struct snd_soc_component *component);
	int (*suspend)(struct snd_soc_component *component);
	int (*resume)(struct snd_soc_component *component);

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

`struct snd_soc_component`
```c
struct snd_soc_component {
	const char *name;
	int id;
	const char *name_prefix;
	struct device *dev;
	/* 该component绑定的snd_soc_card声卡 */
	struct snd_soc_card *card;

	unsigned int active;

	unsigned int suspended:1; /* is in suspend PM state */

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

`struct snd_soc_pcm_runtime`为每一个dai link创建一个pcm runtime
```c
// include/sound/soc.h
/* SoC machine DAI configuration, glues a codec and cpu DAI together */
struct snd_soc_pcm_runtime {
	struct device *dev;
	// 对应machine驱动中创建的ASoC声卡
	struct snd_soc_card *card;
	// 对应ASoC声卡中其中的一个dai link
	struct snd_soc_dai_link *dai_link;
	struct snd_pcm_ops ops;

	unsigned int params_select; /* currently selected param for dai link */

	/* Dynamic PCM BE runtime data */
	struct snd_soc_dpcm_runtime dpcm[2];
	/* stream关闭后经过pmdown_time后DAPM才掉电，单位毫秒，默认是5S */
	long pmdown_time;

	/* runtime devices */
	/* 指向所属的pcm */
	struct snd_pcm *pcm;
	struct snd_compr *compr;

	/*
	 * dais = cpu_dai + codec_dai
	 * see
	 *	soc_new_pcm_runtime()
	 *	asoc_rtd_to_cpu()
	 *	asoc_rtd_to_codec()
	 */
	struct snd_soc_dai **dais;
	/* runtime中codec的数量，从snd_soc_card中获得 */
	unsigned int num_codecs;
	/* runtime中cpu的数量，从snd_soc_card中获得 */
	unsigned int num_cpus;

	struct snd_soc_dapm_widget *playback_widget;
	struct snd_soc_dapm_widget *capture_widget;

	struct delayed_work delayed_work;
	void (*close_delayed_work_func)(struct snd_soc_pcm_runtime *rtd);
#ifdef CONFIG_DEBUG_FS
	struct dentry *debugfs_dpcm_root;
#endif

	unsigned int num; /* 0-based and monotonic increasing */
	/* 通过list节点链接到snd_sco_card->rtd_lists */
	struct list_head list; /* rtd list of the soc card */

	/* function mark */
	struct snd_pcm_substream *mark_startup;
	struct snd_pcm_substream *mark_hw_params;
	struct snd_pcm_substream *mark_trigger;
	struct snd_compr_stream  *mark_compr_startup;

	/* bit field */
	unsigned int pop_wait:1;
	unsigned int fe_compr:1; /* for Dynamic PCM */

	/* dai link中所有的components，cpu dai和codec都看做是一个component */
	int num_components;
	/* 指向runtime中所有的component，按照cpu, codec, platform的顺序 */
	struct snd_soc_component *components[]; /* CPU/Codec/Platform */
};
```

`struct snd_pcm`
```c
// include/sound/pcm.h
struct snd_pcm {
	/* 指向pcm_runtime->card->snd_card */
	struct snd_card *card;
	struct list_head list;
	/* 这里一般情况下就是rtd->num的值 */
	int device; /* device number */
	unsigned int info_flags;
	unsigned short dev_class;
	unsigned short dev_subclass;
	char id[64];
	char name[80];
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

`struct snd_pcm_str`
```c
struct snd_pcm_str {
	/* stream方向 */
	int stream;				/* stream (direction) */
	/* 指向所属pcm */
	struct snd_pcm *pcm;
	/* -- substreams -- */
	unsigned int substream_count;
	unsigned int substream_opened;
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

`struct snd_pcm_substream`
```c
struct snd_pcm_substream {
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

`struct snd_dma_buffer`记录dma内存分配的地址
```c
/*
 * info for buffer allocation
 */
struct snd_dma_buffer {
	struct snd_dma_device dev;	/* device type */
	unsigned char *area;	/* virtual pointer */
	dma_addr_t addr;	/* physical address */
	size_t bytes;		/* buffer size in bytes */
	void *private_data;	/* private for allocator; don't touch */
};
```


`struct snd_soc_dai_driver`dai的驱动结构体，例如phytium i2s中定义的dai driver实例`struct snd_soc_dai_driver phytium_i2s_dai`
```c
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
	/* dai_driver的名字 */
    const char *name;
    unsigned int id;
    unsigned int base;
    struct snd_soc_dobj dobj;

    /* DAI driver callbacks */
    int (*probe)(struct snd_soc_dai *dai);
    int (*remove)(struct snd_soc_dai *dai);
    int (*suspend)(struct snd_soc_dai *dai);
    int (*resume)(struct snd_soc_dai *dai);
    /* compress dai */
    int (*compress_new)(struct snd_soc_pcm_runtime *rtd, int num);
    /* Optional Callback used at pcm creation*/
    int (*pcm_new)(struct snd_soc_pcm_runtime *rtd,
            struct snd_soc_dai *dai);
    /* DAI is also used for the control bus */
    bool bus_control;

    /* ops */
    const struct snd_soc_dai_ops *ops;
    const struct snd_soc_cdai_ops *cops;

    /* DAI capabilities */
    struct snd_soc_pcm_stream capture;
    struct snd_soc_pcm_stream playback;
    unsigned int symmetric_rates:1;
    unsigned int symmetric_channels:1;
    unsigned int symmetric_samplebits:1;

    /* probe ordering - for components with runtime dependencies */
	/* probe的顺序，在snd_soc_pcm_dai_probe()被用到 */
    int probe_order;
    int remove_order;
}
```

`struct snd_soc_dapm_context`
```c
// include/sound/soc-dapm.h

/*
 * Bias levels
 *
 * @ON:      Bias is fully on for audio playback and capture operations.
 * @PREPARE: Prepare for audio operations. Called before DAPM switching for
 *           stream start and stop operations.
 * @STANDBY: Low power standby state when no playback/capture operations are
 *           in progress. NOTE: The transition time between STANDBY and ON
 *           should be as fast as possible and no longer than 10ms.
 * @OFF:     Power Off. No restrictions on transition times.
 */
enum snd_soc_bias_level {
	SND_SOC_BIAS_OFF = 0,
	SND_SOC_BIAS_STANDBY = 1,
	SND_SOC_BIAS_PREPARE = 2,
	SND_SOC_BIAS_ON = 3,
};

/* DAPM context */
struct snd_soc_dapm_context {
	enum snd_soc_bias_level bias_level;
	unsigned int idle_bias_off:1; /* Use BIAS_OFF instead of STANDBY */
	/* Go to BIAS_OFF in suspend if the DAPM context is idle */
	unsigned int suspend_bias_off:1;

	struct device *dev; /* from parent - for debug */
	struct snd_soc_component *component; /* parent component */
	struct snd_soc_card *card; /* parent card */

	/* used during DAPM updates */
	enum snd_soc_bias_level target_bias_level;
	struct list_head list;

	struct snd_soc_dapm_wcache path_sink_cache;
	struct snd_soc_dapm_wcache path_source_cache;

#ifdef CONFIG_DEBUG_FS
	struct dentry *debugfs_dapm;
#endif
};
```

`struct snd_soc_dai`
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

	/* parent platform/codec */
	struct snd_soc_component *component;

	/* CODEC TDM slot masks and params (for fixup) */
	unsigned int tx_mask;
	unsigned int rx_mask;

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

`struct snd_soc_dai_link`结构体，链接cpu dai和codec，在machine驱动中描述
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
	struct snd_soc_dai_link_component *cpus;
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
	unsigned int playback_only:1;
	unsigned int capture_only:1;

	/* Keep DAI active over suspend */
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

`struct snd_kcontrol_new`
```c
// inlcude/sound/control.h
struct snd_kcontrol_new {
	snd_ctl_elem_iface_t iface;	/* interface identifier */
	unsigned int device;		/* device/client number */
	unsigned int subdevice;		/* subdevice (substream) number */
	/* control的名字 */
	const char *name;		/* ASCII name of item */
	unsigned int index;		/* index of item */
	unsigned int access;		/* access rights */
	unsigned int count;		/* count of same elements */
	/* 回调函数，用于获取control的详细信息 */
	snd_kcontrol_info_t *info;
	/* 回调函数，用于读取control的当前值 */
	snd_kcontrol_get_t *get;
	/* 回调函数，用于把应用程序的控制值设置到control中 */
	snd_kcontrol_put_t *put;
	union {
		snd_kcontrol_tlv_rw_t *c;
		const unsigned int *p;
	} tlv;
	unsigned long private_value;
};
```

widget的类型枚举
```c
/* dapm widget types */
enum snd_soc_dapm_type {
	snd_soc_dapm_input = 0,		/* input pin */
	snd_soc_dapm_output,		/* output pin */
	snd_soc_dapm_mux,			/* selects 1 analog signal from many inputs */
	snd_soc_dapm_demux,			/* connects the input to one of multiple outputs */
	snd_soc_dapm_mixer,			/* mixes several analog signals together */
	snd_soc_dapm_mixer_named_ctl,		/* mixer with named controls */
	snd_soc_dapm_pga,			/* programmable gain/attenuation (volume) */
	snd_soc_dapm_out_drv,			/* output driver */
	snd_soc_dapm_adc,			/* analog to digital converter */
	snd_soc_dapm_dac,			/* digital to analog converter */
	snd_soc_dapm_micbias,		/* microphone bias (power) - DEPRECATED: use snd_soc_dapm_supply */
	snd_soc_dapm_mic,			/* microphone */
	snd_soc_dapm_hp,			/* headphones */
	snd_soc_dapm_spk,			/* speaker */
	snd_soc_dapm_line,			/* line input/output */
	snd_soc_dapm_switch,		/* analog switch */
	snd_soc_dapm_vmid,			/* codec bias/vmid - to minimise pops */
	snd_soc_dapm_pre,			/* machine specific pre widget - exec first */
	snd_soc_dapm_post,			/* machine specific post widget - exec last */
	snd_soc_dapm_supply,		/* power/clock supply */
	snd_soc_dapm_pinctrl,		/* pinctrl */
	snd_soc_dapm_regulator_supply,	/* external regulator */
	snd_soc_dapm_clock_supply,	/* external clock */
	snd_soc_dapm_aif_in,		/* audio interface input */
	snd_soc_dapm_aif_out,		/* audio interface output */
	snd_soc_dapm_siggen,		/* signal generator */
	snd_soc_dapm_sink,
	snd_soc_dapm_dai_in,		/* link to DAI structure */
	snd_soc_dapm_dai_out,
	snd_soc_dapm_dai_link,		/* link between two DAI structures */
	snd_soc_dapm_kcontrol,		/* Auto-disabled kcontrol */
	snd_soc_dapm_buffer,		/* DSP/CODEC internal buffer */
	snd_soc_dapm_scheduler,		/* DSP/CODEC internal scheduler */
	snd_soc_dapm_effect,		/* DSP/CODEC effect component */
	snd_soc_dapm_src,		/* DSP/CODEC SRC component */
	snd_soc_dapm_asrc,		/* DSP/CODEC ASRC component */
	snd_soc_dapm_encoder,		/* FW/SW audio encoder component */
	snd_soc_dapm_decoder,		/* FW/SW audio decoder component */

	/* Don't edit below this line */
	SND_SOC_DAPM_TYPE_COUNT
};
```

`struct snd_soc_dapm_route`的定义
```c
/*
 * DAPM audio route definition.
 *
 * Defines an audio route originating at source via control and finishing
 * at sink.
 */
struct snd_soc_dapm_route {
	/* 要连接的控件的名字 */
	const char *sink;
	/* 连接开关的名字 */
	const char *control;
	/* 要连接的控件的名字 */
	const char *source;

	/* Note: currently only supported for links where source is a supply */
	int (*connected)(struct snd_soc_dapm_widget *source,
			 struct snd_soc_dapm_widget *sink);

	struct snd_soc_dobj dobj;
};
```


### machine 驱动
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

`struct snd_soc_card`注册过程
```c
pmdk_sound_probe()
    devm_snd_soc_register_card(&pdev->dev, card);
        snd_soc_register_card(card);
            snd_soc_bind_card(card);
                // 
                snd_soc_dapm_init(card->dapm, card, NULL);

```

`snd_soc_dapm_init()`

### plarform驱动
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
	void __iomem *regs;
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


### 音频数据传递流程
linux用户空间应用程序通过声卡驱动程序和linux内核ALSA框架导出的PCM设备文件，如`/dev/snd/pcmC0D0c`和`/dev/snd/pcmC0D0p`等，与linux内核音频设备驱动程序和音频硬件进行数据传递，PCM设备文件的`file_operations`定义如下
```c
// sound/core/pcm_native.c
/*
 *  Register section
 */

const struct file_operations snd_pcm_f_ops[2] = {
	{
		.owner =		THIS_MODULE,
		.write =		snd_pcm_write,
		.write_iter =		snd_pcm_writev,
		.open =			snd_pcm_playback_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_poll,
		.unlocked_ioctl =	snd_pcm_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	},
	{
		.owner =		THIS_MODULE,
		.read =			snd_pcm_read,
		.read_iter =		snd_pcm_readv,
		.open =			snd_pcm_capture_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_poll,
		.unlocked_ioctl =	snd_pcm_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	}
};
```

大多数情况下，音频设备会同时提供播放和录制功能，用于播放和录制的PCM设备文件是一起导出的，播放和录制的PCM设备文件的文件操作也是一起定义的。其中播放的索引为`SNDRV_PCM_STREAM_PLAYBACK`，其值为0；录制的索引为`SNDRV_PCM_STREAM_CAPTURE`，其值为1。

linux用户空间有alsa-lib和tinyalsa等库用于与linux内核alsa框架交互，alsa-lib和tinyalsa等库主要通过ioctl命令与linux内核ALSA框架交互。PCM设备文件的`ioctl`操作函数定义如下：
```c
// sound/core/pcm_native.c
static int snd_pcm_common_ioctl(struct file *file,
				 struct snd_pcm_substream *substream,
				 unsigned int cmd, void __user *arg)
{
	struct snd_pcm_file *pcm_file = file->private_data;
	int res;

	if (PCM_RUNTIME_CHECK(substream))
		return -ENXIO;

	res = snd_power_wait(substream->pcm->card);
	if (res < 0)
		return res;

	switch (cmd) {
	case SNDRV_PCM_IOCTL_PVERSION:
		return put_user(SNDRV_PCM_VERSION, (int __user *)arg) ? -EFAULT : 0;
	case SNDRV_PCM_IOCTL_INFO:
		return snd_pcm_info_user(substream, arg);
	case SNDRV_PCM_IOCTL_TSTAMP:	/* just for compatibility */
		return 0;
	case SNDRV_PCM_IOCTL_TTSTAMP:
		return snd_pcm_tstamp(substream, arg);
	case SNDRV_PCM_IOCTL_USER_PVERSION:
		if (get_user(pcm_file->user_pversion,
			     (unsigned int __user *)arg))
			return -EFAULT;
		return 0;
	case SNDRV_PCM_IOCTL_HW_REFINE:
		return snd_pcm_hw_refine_user(substream, arg);
	case SNDRV_PCM_IOCTL_HW_PARAMS:
		return snd_pcm_hw_params_user(substream, arg);
	case SNDRV_PCM_IOCTL_HW_FREE:
		return snd_pcm_hw_free(substream);
	case SNDRV_PCM_IOCTL_SW_PARAMS:
		return snd_pcm_sw_params_user(substream, arg);
	case SNDRV_PCM_IOCTL_STATUS32:
		return snd_pcm_status_user32(substream, arg, false);
	case SNDRV_PCM_IOCTL_STATUS_EXT32:
		return snd_pcm_status_user32(substream, arg, true);
	case SNDRV_PCM_IOCTL_STATUS64:
		return snd_pcm_status_user64(substream, arg, false);
	case SNDRV_PCM_IOCTL_STATUS_EXT64:
		return snd_pcm_status_user64(substream, arg, true);
	case SNDRV_PCM_IOCTL_CHANNEL_INFO:
		return snd_pcm_channel_info_user(substream, arg);
	case SNDRV_PCM_IOCTL_PREPARE:
		return snd_pcm_prepare(substream, file);
	case SNDRV_PCM_IOCTL_RESET:
		return snd_pcm_reset(substream);
	case SNDRV_PCM_IOCTL_START:
		return snd_pcm_start_lock_irq(substream);
	case SNDRV_PCM_IOCTL_LINK:
		return snd_pcm_link(substream, (int)(unsigned long) arg);
	case SNDRV_PCM_IOCTL_UNLINK:
		return snd_pcm_unlink(substream);
	case SNDRV_PCM_IOCTL_RESUME:
		return snd_pcm_resume(substream);
	case SNDRV_PCM_IOCTL_XRUN:
		return snd_pcm_xrun(substream);
	case SNDRV_PCM_IOCTL_HWSYNC:
		return snd_pcm_hwsync(substream);
	case SNDRV_PCM_IOCTL_DELAY:
	{
		snd_pcm_sframes_t delay;
		snd_pcm_sframes_t __user *res = arg;
		int err;

		err = snd_pcm_delay(substream, &delay);
		if (err)
			return err;
		if (put_user(delay, res))
			return -EFAULT;
		return 0;
	}
	case __SNDRV_PCM_IOCTL_SYNC_PTR32:
		return snd_pcm_ioctl_sync_ptr_compat(substream, arg);
	case __SNDRV_PCM_IOCTL_SYNC_PTR64:
		return snd_pcm_sync_ptr(substream, arg);
#ifdef CONFIG_SND_SUPPORT_OLD_API
	case SNDRV_PCM_IOCTL_HW_REFINE_OLD:
		return snd_pcm_hw_refine_old_user(substream, arg);
	case SNDRV_PCM_IOCTL_HW_PARAMS_OLD:
		return snd_pcm_hw_params_old_user(substream, arg);
#endif
	case SNDRV_PCM_IOCTL_DRAIN:
		return snd_pcm_drain(substream, file);
	case SNDRV_PCM_IOCTL_DROP:
		return snd_pcm_drop(substream);
	case SNDRV_PCM_IOCTL_PAUSE:
		return snd_pcm_pause_lock_irq(substream, (unsigned long)arg);
	case SNDRV_PCM_IOCTL_WRITEI_FRAMES:
	case SNDRV_PCM_IOCTL_READI_FRAMES:
		return snd_pcm_xferi_frames_ioctl(substream, arg);
	case SNDRV_PCM_IOCTL_WRITEN_FRAMES:
	case SNDRV_PCM_IOCTL_READN_FRAMES:
		return snd_pcm_xfern_frames_ioctl(substream, arg);
	case SNDRV_PCM_IOCTL_REWIND:
		return snd_pcm_rewind_ioctl(substream, arg);
	case SNDRV_PCM_IOCTL_FORWARD:
		return snd_pcm_forward_ioctl(substream, arg);
	}
	pcm_dbg(substream->pcm, "unknown ioctl = 0x%x\n", cmd);
	return -ENOTTY;
}
```





