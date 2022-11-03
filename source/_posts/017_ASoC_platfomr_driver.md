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

+ ASoC Platform驱动包括音频DMA引擎驱动程序（PCM DMA）和数字音频接口驱动程序（CPU DAI）

+ CPU DAI：在嵌入式系统里面通常指CPU的I2S或PCM总线控制器，对于Playback过程来说，负责将音频数据从I2S TX FIFO搬运到CODEC，cpu_dai通过snd_soc_register_dai()来注册
