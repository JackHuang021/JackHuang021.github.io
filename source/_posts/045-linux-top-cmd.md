---
title: 045_linux_top_cmd
tags:
  - Linux
  - top
categories: Linux
abbrlink: f4a20648
date: 2023-07-19 10:02:04
---

top能够实时显示系统中各个进程的资源占用情况，类似于windows的任务管理器
![](https://raw.githubusercontent.com/JackHuang021/images/master/
<!-- more -->

### 1. top参数含义
20230719103527.png)
前5行是系统整体的统计信息：
+ 第1行是任务队列信息，同uptime的执行结果，其内容如下
    ```bash
    # 当前时间          系统运行时间    当前登录用户数      系统负载
    top - 10:37:51 up  1:53,  1 user,  load average: 0.73, 0.96, 0.76
    ```
+ 第2 3行为进程和CPU的统计信息
    ```bash
    # 总进程数      正在运行的进程数    睡眠的进程数    停止的进程数    僵尸进程数
    Tasks: 409 total,   1 running, 403 sleeping,   0 stopped,   5 zombie
    # us 用户空间占用CPU百分比
    # sy 内核空间占用CPU百分比
    # ni 用户进程空间内改变过优先级的进程占用CPU百分比
    # id CPU空闲百分比
    # wa 等待输入输出的CPU时间占用百分比
    # hi 硬件中断占用百分比
    # si 软中断占用百分比
    # st 虚拟机占用百分比
    %Cpu(s):  0.5 us,  0.2 sy,  0.0 ni, 99.2 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
    ```

+ 最后两行为内存信息
    ```bash
    # 物理内存总量      空闲的物理内存总量      已使用的物理内存总量        用作内核缓存的内存量
    MiB Mem :  15753.0 total,    276.8 free,   5090.9 used,  10385.4 buff/cache
    # 交换分区内存总量      空闲交换分区总量        已使用交换分区内存总量
    MiB Swap:   4096.0 total,   4091.2 free,      4.8 used.   9469.2 avail Mem 
    ```

进程信息统计区显示了各个进程的详细信息
+ PID：进程ID
+ USER：进程所有者的用户id
+ PR：进程优先级
+ NI：nice值，负值表示高优先级，正值表示低优先级
+ VIRT：进程使用的虚拟内存总量，单位kb
+ RES：进程使用的，未被换出的物理内存大小，单位kb
+ SHR：共享内存大小，单位kb
+ S：进程状态(D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程)
+ %CPU：上次更新到本次的CPU时间占用百分比
+ %MEM：进程使用的物理内存百分比
+ TIME+：进程使用的CPU时间总计
+ COMMAND：进程名称




