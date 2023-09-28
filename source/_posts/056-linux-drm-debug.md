---
title: Linux 显示相关问题调试
date: 2023-09-26 09:21:23
tags:
  - Linux
  - drm
categories: Linux
---

针对xorg的显示问题排查步骤

1. 查看xorg log，`cat /var/log/Xorg.0.log`，从log信息中查找(WW)和(EE)相关log

2. 查看drm sysfs目录信息`/sys/class/drm`，

3. 打开drm驱动调试信息，增加内核启动参数`drm.debug=0x1f`