---
title: linux 桌面问题排故
tags:
---

## 了解当前桌面环境

1. 查看当前的显示协议是 Wayland 还是 Xorg: `echo $XDG_SESSION_TYPE`

2. 查看当前的显示管理器：`echo /etc/X11/default-display-manager`

## 桌面相关测试

1. glmark2 （apt install glmark2）测试

2. drm 显示驱动测试 modetest (apt install libdrm-tests)

## 日志查看


