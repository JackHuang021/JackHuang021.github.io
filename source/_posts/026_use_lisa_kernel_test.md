---
title: LISA内核测试工具使用
tags:
  - Linux
  - LISA
categories: Linux
abbrlink: 474f3fa1
---

### 介绍
LISA(Linux Intergrated/Interactive System Analysis)，支持对Linux内核进行回归测试和交互式分析，LISA的目标是帮助Linux内核开发人员衡量内核核心部分修改的影响，重点关注任务调度（例如EAS）、电源管理和热管理框架

<!-- more -->
LISA作为一个“Host/Target”模型，其本身运行在宿主机上，使用devlib工具通过ssh、ADB、telnet和目标板进行交互，对于目标板只要是基于Linux内核系统就行

LISA提供了工作负载，并在目标板上运行这些工作负载（使用rt-app），同时从目标板收集Trace文件（使用systrace或者ftrace）并进行解析，分析

![](https://raw.githubusercontent.com/JackHuang021/images/master/20221215093113.png)

### 安装
#### Host安装
使用v3.1.0版本
```bash
# clone源码，基于v3.1.0 tag创建分支
git clone https://github.com/ARM-software/lisa.git
git branch v3.1.0_dev v3.1.0
git checkout v3.1.0_dev
# This will provide a more accurate changelog when building the doc
git fetch origin refs/notes/changelog
# 安装需要的软件包
sudo ./install_base.sh --install-all
# On the first run, it will take care of creating a Python venv and populating it
source init_env
```
#### Target安装
