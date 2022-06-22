---
title: 龙芯2K1000内核源码编译
author: Jack
tags:
  - PMON
  - Loongson
categories:
  - Linux
abbrlink: 67263a86
date: 2022-06-22 08:35:41
---

#### 解压PMON源码*pmon-loongson3.tar.gz*
```
tar -xvf pmon-loongson3-nd-33j.tar.gz
```
#### 配置交叉编译环境
+ 解压交叉编译工具*gcc-4.4.0-pmon.tgz*，配置环境
<!-- more -->
```
tar -xvf gcc-4.4.0-pmon.tgz
```
编辑/etc/profile，配置交叉编译工具路径
```
export PATH=$PATH:/home/jack/loongson/tools/gcc-4.4.0-pmon/bin
export LD_LIBRARY_PATH=/home/jack/loongson/tools/gcc-4.4.0-pmon/lib:$LD_LIBRARY_PATH

```
+ 安装makedepend，`sudo apt-get install xutils-dev `
+ 进入源码目录编译安装pmoncfg
```
sudo apt-get install bison flex build-essential patch
cd tools/pmoncfg 
make 
sudo cp pmoncfg /usr/bin
```

#### 编译PMON源码
进入PMON源码目录，若只需要修改PMON，可在原来的项目目录基础上进行编译，源码内有脚本文件*build.sh*，输入格式: `./build.sh [cputype] [proID]`，比如龙芯2K1000，项目ID hm19047，输入`./build.sh ls2k hm19047`即可开始编译，编译结果在*zloader.ls2k-hm19047*目录中，会生成*gzrom-dtb.bin*，将该文件烧录进flash即可。

#### PMON网络烧录
+ 主机IP地址为192.168.0.100，搭建tftp服务器
+ 开机按C键进入PMON下，设置IP地址`ifconfig syn0 192.168.0.10`
+ 烧写PMON`load -rf 0xbfc00000 tftp://192.168.0.100/gzrom-dtb.bin`
