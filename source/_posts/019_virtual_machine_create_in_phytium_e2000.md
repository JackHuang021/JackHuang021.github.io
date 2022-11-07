---
title: phytium E2000搭建aarch64虚拟机
author: Jack
tags:
  - Linux
  - qemu
  - 虚拟机
categories:
  - Linux
abbrlink: 4ed0329a
---

> 最近需要在E2000平台上测试虚拟化功能，需要搭建一个arm64虚拟机用于测试，记录一下搭建过程

### 系统环境
+ E2000Q CPU，Linux内核版本4.19.246，Ubuntu 20.04
+ 清华软件源
+ 交叉编译工具，`gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu`
<!-- more -->

### qemu安装
+ 安装qemu，`apt-get install qemu-system-arm`
+ 安装aarch64 uefi固件，`apt-get install qemu-efi-aarch64`

### 根文件系统制作
+ 下载BusyBox源码，[官网下载地址](https://busybox.net/downloads/)，这里下载`1.33.1`版本
+ 解压源码，进入源码目录，进行编译
    ```bash
    tar -xvf busybox-1.33.1.tar.bz2
    cd busybox-1.33.1
    export ARCH=arm64
    export CROSS_COMPILE=aarch64-linux-gnu-
    make menuconfig
    ```
    选择静态编译  
    ![](https://raw.githubusercontent.com/JackHuang021/images/master/20221107093137.png)
    ```bash
    make -j12
    make install
    ```
    编译完成后，源码目录`_install`下会生成根文件系统

+ 完善根文件系统
    + 创建其他目录`mkdir dev etc lib`
    + 创建`/etc/profile`文件
        ```bash
        #!/bin/sh
        export HOSTNAME=linux
        export USER=root
        export HOME=/home
        export PS1="[$USER@$HOSTNAME \W]\# "
        PATH=/bin:/sbin:/usr/bin:/usr/sbin
        LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
        export PATH LD_LIBRARY_PATH
        ```
    + 创建`/etc/inittab`文件
        ```
        ::sysinit:/etc/init.d/rcS
        ::respawn:-/bin/sh
        ::askfirst:-/bin/sh
        ::ctrlaltdel:/bin/umount -a -r
        ```
    + 创建`/etc/init.d/rcS`文件
        ```bash
        mkdir -p /sys
        mkdir -p /tmp
        mkdir -p /proc
        mkdir -p /mnt
        /bin/mount -a
        mkdir -p /dev/pts
        mount -t devpts devpts /dev/pts
        echo /sbin/mdev > /proc/sys/kernel/hotplug
        mdev -s
        ```
    + 制作`/dev`目录下必要的文件，`sudo mknod console c 5 1`

+ 编译内核
    + Linux源码下载地址，[官网下载地址](https://mirrors.edge.kernel.org/pub/linux/kernel/)
    + 这里下载`4.19.262`版本，解压内核源码，将之前生成的根文件目录到内核源码目录
        ```bash
        tar -xvf linux-4.19.262.tar.xz
        cd linux-4.19.246
        cp -a ../busybox-1.33.1/_install  rootfs -a
        export ARCH=arm64
        export CROSS_COMPILE=aarch64-linux-gnu-
        make defconfig
        make menucofig
        ```
        设置Initramfs支持
        ![](https://raw.githubusercontent.com/JackHuang021/images/master/20221107112153.png)
        ```
        make all -j12
        ```
    + 将`arch/arm64/boot/Image`拷贝到E2000开发板上

### Qemu启动虚拟机
    ```bash
    qemu-system-aarch64 \
	-enable-kvm \
	-m 2048 \
	-smp 4 \
	-cpu host \
	-M virt,gic-version=3 \
	-kernel Image \
	-append "rdinit=/linuxrc console=ttyAMA0 loglevel=8" \
	-net none \
	-nographic 
    ```



