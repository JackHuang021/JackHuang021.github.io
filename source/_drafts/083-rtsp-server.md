---
title: 083_rtsp_server
tags:
---

## 简介

v4l2hrtspserver 仓库地址：[https://github.com/mpromonet/v4l2rtspserver](https://github.com/mpromonet/v4l2rtspserver)

## v4l2 相关知识

### 像素格式定义

v4l2 中的 fourcc （Four Character Code） 格式是用来表示像素格式的一种方式，广泛用于视频采集和图像处理领域。

fourcc 是一个32位的标识符，由四个 ascii 字符组成，表示某种具体的图像像素格式

```c
// /usr/include/linux/videodev2.h
/*  Four-character-code (FOURCC) */
#define v4l2_fourcc(a, b, c, d)\
	((__u32)(a) | ((__u32)(b) << 8) | ((__u32)(c) << 16) | ((__u32)(d) << 24))

// 例如
v4l2_fourcc('Y', 'U', 'Y', 'V')  // 代表 YUYV 格式
```

| 格式代号   | 说明             | 描述类型       |
| ------ | -------------- | ---------- |
| `YUYV` | YUYV 4:2:2     | 半平面 YUV 格式 |
| `MJPG` | Motion JPEG    | 压缩格式       |
| `H264` | H.264 视频流      | 压缩格式       |
| `NV12` | YUV 4:2:0, 半平面 | 半平面 YUV 格式 |
| `RGB3` | RGB24          | 每像素 3 字节   |
| `GREY` | 灰度图像，8-bit     | 单通道灰度      |

可以使用 v4l2-ctl 查看视频设备支持的 fourcc 格式：

```bash
v4l2-ctl --list-formats
```
