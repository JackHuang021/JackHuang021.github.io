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
+ 一个声卡可以有多个设备，一个设备可以有多个播放\录音通道，每个设备节点都对应一个fops，


### ASoC驱动框架
+ ASoC是建立在标准ALSA驱动层上，对底层的ALSA框架封装了一层，为了更好的支持嵌入式CPU和音频编解码设备的一套软件体系，ASoC驱动主要由Platform驱动、Codec驱动、Machine驱动组成。


+ CPU DAI：在嵌入式系统中通常指CPU的I2S或PCM总线控制器
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
        int probe_order;
        int remove_order;
    }
    ```
