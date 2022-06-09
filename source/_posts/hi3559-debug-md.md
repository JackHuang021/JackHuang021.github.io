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

#### 驱动调试
+ 参照正点原子教程搭建驱动调试环境，配置的*Makefile*内容如下
    ```
    KERNEL_DIR = /home/jack/hisi/minimum_system/linux-4.9.y_multi-core
    CURRENT_PATH = $(shell pwd)
    obj-m := gpio-pca953x.o

    build: kernel_modules

    kernel_modules: 
    $(MAKE) ARCH=arm64 CROSS_COMPILE=aarch64-himix100-linux- -C $(KERNEL_DIR) M=$(CURRENT_PATH) modules

    clean: 
    $(MAKE) ARCH=arm64 CROSS_COMPILE=aarch64-himix100-linux- -C $(KERNEL_DIR) M=$(CURRENT_PATH) clean
    ```
+ *KERNEL_DIR*表示Linux内核源码目录，使用绝对路径
+ *CURRENT_PATH*表示当前路径，直接使用*pwd*来获取当前路径
+ *obj-m*表示将这个c文件编译为ko模块
+ *modules*表示编译模块，*-C*表示将当前的工作目录切换到指定目录中， *M*表示模块源码目录
+ *make modules*命令中加入 M=dir 以后程序会自动到指定的 dir 目录中读取模块的源码并将其编译为.ko 文件
