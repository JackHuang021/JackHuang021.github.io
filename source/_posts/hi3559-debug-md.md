---
title: HI3559AV100调试记录
author: Jack
tags:
  - Hi3559AV100 Linux
categories:
  - Linux
abbrlink: 7803046f
date: 2022-06-09 10:02:24
---

#### 内核编译与烧写
+ 内核版本4.9.37，Linux内核源码如下
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609101727.png)
<!-- more -->
+ 编译`make ARCH=arm64 CROSS_COMPILE=aarch64-himix100-linux- uImage -j12`，编译完成结果如下，编译完成后会在arch/arm64/boot/生成UImage
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609101926.png)
+ arm-trusted-firmware目录中运行mk.sh生成fip.bin
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609110821.png)
+ 主机搭建tftp服务器，将fip.bin拷贝到共享目录
+ 进入uboot，配置ethact ipaddr serverip环境变量
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609111122.png)
+ 从tftp加载内核，测试内核是否能启动、
    ```
    tftp fip.bin 0x42000000
    bootm 0x42000000
    ```
+ 烧写内核到emmc
    ```
    tftp fip.bin 0x42000000
    mmc dev 0 0
    mmc write 0 0x42000000 800 4800
    ```

