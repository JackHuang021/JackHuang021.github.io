---
title: Ubuntu键盘快捷键映射
tags:
---

### 1. 问题描述

在Ubuntu中某些快捷键（亮度调节等）无法起作用，使用evtest查看键盘输入事件，发现按下这些快捷键的时候有输入事件报上来，例如：

```bash
Event: time 1752779445.406990, type 4 (EV_MSC), code 4 (MSC_SCAN), value c1
Event: time 1752779445.406990, -------------- SYN_REPORT ------------
Event: time 1752779445.529960, type 4 (EV_MSC), code 4 (MSC_SCAN), value c1
Event: time 1752779445.529960, -------------- SYN_REPORT ------------
Event: time 1752779445.826293, type 4 (EV_MSC), code 4 (MSC_SCAN), value c1
```

从 evtest 输出可以看到，按下亮度调节按键时系统确实检测到了 EV_MSC / MSC_SCAN 类型的事件，value c1 是这个按键的扫描码，但并没有看到 EV_KEY 类型的事件，这表明这个按键可能没有被内核识别为标准的功能键（比如 KEY_BRIGHTNESS_UP / KEY_BRIGHTNESS_DOWN），导致系统无法正常响应亮度调节。

### 2. 解决方法 使用 udev/hwdb 添加按键映射

udev提供了一个称为"hwdb"的内置函数，用于维护 `/etc/udev/hwdb.bin` 中的硬件数据库索引。 数据库是从位于`/usr/lib/udev/hwdb.d/`、 `/run/udev/hwdb.d/` 和 `/etc/udev/hwdb.d/` 目录中的扩展名为.hwdb的文件编译而成的。 默认的 "扫描码到键码" 映射文件是 `/usr/lib/udev/hwdb.d/60-keyboard.hwdb`。

.hwdb 文件可以包含不同键盘的多个映射块，也可以将一个块应用于多个键盘。 evdev: 前缀用于将块与硬件进行匹配。

扫描码映射至键位码的详细说明可以参考archlinux的官方文档：[https://wiki.archlinux.org/title/Map_scancodes_to_keycodes](https://wiki.archlinux.org/title/Map_scancodes_to_keycodes)

#### 2.1 操作步骤，以飞腾笔记本为例

1. 新增一个hwdb文件`/etc/udev/hwdb.d/90-custom-keyboard.hwdb`，添加飞腾笔记本AT键盘DMI数据匹配：

	```shell
	##########################################
	# Phytium
	##########################################

	hwplatformPhytium:evdev:atkbd:dmi:bvn*:bvr*:bd*:svn*:pn*:pvr*:rvn*
	KEYBOARD_KEY_bd=rfkill             # RFkill
	KEYBOARD_KEY_d4=screenlock         # screenlock
	KEYBOARD_KEY_bf=f21                # Touchpad on/off
	KEYBOARD_KEY_c2=f20                # MIC On/Off
	KEYBOARD_KEY_81=brightnessdown     # Brightness down
	KEYBOARD_KEY_82=brightnessup       # Brightness up
	KEYBOARD_KEY_be=battery            # fn+Q
	KEYBOARD_KEY_83=f21                # Touchpad on/off
	KEYBOARD_KEY_84=rfkill             # RFKill
	KEYBOARD_KEY_85=f22                # Touchpad On
	KEYBOARD_KEY_86=f23                # Touchpad Off
	KEYBOARD_KEY_87=brightness_toggle  # Display LCD on/off
	KEYBOARD_KEY_88=switchvideomode    # Switch Display Mode
	KEYBOARD_KEY_8a=f20                # MIC On/Off
	KEYBOARD_KEY_8b=camera             # Camera On/Off
	KEYBOARD_KEY_ff=screenlock         # KEY_SCREENLOCK
	```

2. 更新硬件数据库索引： `systemd-hwdb update`

3. 重新加载硬件数据库索引: `udevadm trigger`
