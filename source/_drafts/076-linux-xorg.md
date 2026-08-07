---
title: linux xorg
tags:
---

## 1. x11协议

### 1.1 协议格式

**请求**： 每一个请求由请求头和请求体组成，请求头由4个字节组成，包含一个8位的主操作码和一个16位的请求数据长度（这个长度包括了请求头的长度）。主操作码128-255留给了扩展组件使用，扩展组件的请求里可能包含多个操作码，这时请求的结构会有一些改变。比如，一种典型的用法是，在头的第二个字节里存放一个副操作码(minor opcode)，把长度值移到后面两个字节去。连接上的每个请求都会分配一个序列号，用于回复，错误和事件协议。

**回复**： 回复由4个字节的回复长度字段和32个字节的回复数据组成，回复还要包含对应请求序列号的最低16位（未说明在哪个字段位置）

**错误**： 错误报告的长度为32字节，包含一个8位的错误码，其中128-255预留给了拓展组件，在错误报告中还包含所对应请求的主操作码、副操作码和序列号。

**事件**： 事件的长度为32字节，包含一个8位的事件类型码，其中64-127预留给了拓展组件，事件类型码中会有一位用于标识该事件是否是由客户端发起的，即如果这一事件是因响应客户端的SendEvent请求而发出的，这将被置位

## 2. Xorg显示服务

Xorg是X11协议的一个实现，而X Window System是一个C/S结构的程序，Xorg只是提供了一个X Server，负责底层的操作当你运行一个程序的时候，这个程序会连接到X server上，由X server接收键盘鼠标输入和负责屏幕输出窗口的移动，窗口标题的样式等等。

+ x11协议地址: [https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html)
+ Xorg 仓库地址： [https://gitlab.freedesktop.org/xorg/xserver.git](https://gitlab.freedesktop.org/xorg/xserver.git)

### 2.1 编译Xorg

1. 下载Xorg源码，使用Xorg -version检查系统安装的Xorg版本，切换到对应版本的tag

   ```bash
   git clone https://gitlab.freedesktop.org/xorg/xserver.git
   git checkout xorg-server-21.1.4
   ```

2. 编译Xorg

   ```bash
   apt-get build-dep xorg
   mkdir build && cd build
   meson setup ..
   ninja
   ```

lightdm下xorg运行参数：

```bash
/usr/lib/xorg/Xorg -core :0 -seat seat0 -auth /var/run/lightdm/root/:0 -nolisten tcp vt7 -novtswitch
```

gdm3环境下的xorg运行参数

```bash
/usr/lib/xorg/Xorg vt1 -displayfd 3 -auth /run/user/126/gdm/Xauthority -nolisten tcp -background none -noreset -keeptty -novtswitch -verbose 3
```

### 2.2 Xorg相关的环境变量

1. DISPLAY： 显示服务的标识符，决定X客户端连接哪个X服务器
2. XAUTHORITY： 指定用于 X11 认证的 .Xauthority 文件的路径。这个文件包含了用于验证和授权访问 X11 会话的认证信息。`/home/user/.Xauthority`为默认的认证文件。

### 2.3 Xorg的认证机制

X11 认证机制涉及 X 服务器的访问控制，用于授权和验证哪些用户或主机能够连接到本地 X 服务器并运行图形应用程序。

**.Xauthority文件**： 用于存储X11的认证信息，它保存了允许访问 X 服务器的用户和主机的认证密钥，通常是一个 "magic cookie"。每次启动一个X11服务器时，X11服务器会生成一个随机的认证密钥，并将其存储在一个临时文件中，这个密钥用于验证试图连接到该X11服务器的客户端。

**xauth工具**：xauth 负责管理 .Xauthority 文件，它通过向文件中添加、删除认证密钥来控制哪些用户或主机能够通过认证访问 X 服务器。

Xorg的`-auth`参数用来指定X服务器使用的认证文件，这个文件就包含了认证密钥，这个文件的权限是仅root用户可读写的，可以通过xauth工具来查看它的密钥信息：

```bash
jack@Ubuntu:~$ sudo ls -alh /var/run/lightdm/root/:0
-rw------- 1 root root 51 Nov 21 14:30 /var/run/lightdm/root/:0
jack@Ubuntu:~$ sudo xauth -f /var/run/lightdm/root/:0 list
Ubuntu/unix:0  MIT-MAGIC-COOKIE-1  79681b5184bc62b3fdb5cb8ede265671
```

这个密钥的组成如下：

1. Ubuntu/unix:0： 该字段表明此认证条目是针对本机（Ubuntu/unix）的第一个显示服务（:0）的认证信息。
2. MIT-MAGIC-COOKIE-1： 这是 认证类型。MIT-MAGIC-COOKIE-1 是 X11 认证的标准类型，用于验证客户端连接。
3. 79681b5184bc62b3fdb5cb8ede265671： 认证密钥，通常是 128 位（16 字节）长度的十六进制字符串。这个密钥被用来验证访问该 X 服务器的客户端。

当用户登录进入桌面后，`/home/user/.Xauthority`将会更新为`/var/run/lightdm/root/:0`中的密钥信息，所以在未登录进入桌面前，运行X客户端应用会提示如下的错误`Invalid MIT-MAGIC-COOKIE-1 keyCan't open display :0`，即和X11服务器认证失败，通过xauth工具可以手动将X11的认证密钥添加到`/home/user/.Xauthority`中，这时便可以和X11服务器认证成功。

### 2.4 Xorg在用户态的主要组成部分

1. 核心组件： Xorg server
2. 协议拓展组件，拓展组件的路径为`/usr/lib/xorg/modules/extensions/`
3. 工具：
    + xrandr：管理和调整显示设置
    + xinput：管理输入设备
    + xset：管理X服务器的设置
    + xkbcomp：管理键盘布局
4. 客户端库：
    + libX11：X11协议的实现库，和xorg server进行交互
    + libxcb：X协议的轻量级库

### 2.5 xorg gdb调试

以ubuntu 22.04为例，参考xorg官网xerver debug文档： [https://www.x.org/wiki/Development/Documentation/ServerDebugging/](https://www.x.org/wiki/Development/Documentation/ServerDebugging/)

1. 安装xorg调试符号，参考ubuntu官网教程：[https://ubuntu.com/server/docs/debug-symbol-packages](https://ubuntu.com/server/docs/debug-symbol-packages)

   ```bash
   # 创建/etc/apt/sources.list.d/ddebs.list，加入调试符号软件源
   echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
   deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
   deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | tee -a /etc/apt/sources.list.d/ddebs.list

   apt install ubuntu-dbgsym-keyring
   apt-get update
   apt-get install xserver-xorg-core-dbgsym
   ```

2. gdb调试

   ```bash
   gdb /usr/lib/xorg/Xorg $(pidof Xorg)
   ```

   ![](https://raw.githubusercontent.com/JackHuang021/images/master/20241127104135.png)

命令行模式和桌面模式切换

```bash
sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target
```

## 3. LightDM

LightDM 是一个轻量级的显示管理器（Display Manager），主要用于管理 Linux 和其他 Unix 系统的用户登录会话。显示管理器在系统启动时加载，是用户进入图形桌面环境的入口。LightDM 具有轻量、灵活和模块化的特点，支持多种桌面环境，并允许用户自定义登录界面。

LightDM 仓库地址：[https://github.com/canonical/lightdm](https://github.com/canonical/lightdm)

### 3.1 lightdm-gtk-greeter

lightdm-gtk-greeter 是 LightDM 的一个登录界面插件(greeter)。它使用 GTK 库来提供图形界面，展示用户名、密码输入框、会话选择等控件。

lightdm-gtk-greeter 仓库地址：[https://github.com/Xubuntu/lightdm-gtk-greeter](https://github.com/Xubuntu/lightdm-gtk-greeter)

编译lightdm-gtk-greeter

1. 安装依赖

   ```bash
   sudo apt-get install gobject-introspection xfce4-dev-tools libgtk-3-dev
   ```

2. 配置编译环境，如果遇到缺少的依赖项错误，可以通过安装相应的库来解决

   ```bash
   ./autogen.sh
   ```

3. 编译源码

   ```bash
   make
   ```

## 4. xrandr设置屏幕分辨率流程

xrandr设置分辨率流程涉及到libxrandr、libX11、xserver、libdrm还有linux内核drm驱动，具体的流程如下：
![](https://raw.githubusercontent.com/JackHuang021/images/master/xrandr分辨率设置流程.png)

## 5. Xorg相关问题及解决方法

### 5.1 未登录系统之前运行X11客户端程序报错

在未登录桌面之前，`~/.Xauthority`的密钥还未更新，此时若通过串口或者ssh终端运行X11客户端程序，即使设置了DISPLAY环境变量，也会报`Invalid MIT-MAGIC-COOKIE-1 key`认证错误，需要手动将密钥加到`~/.Xauthority`中：

```bash
export DISPLAY=:0
sudo xauth -f /var/run/lightdm/root/:0 list
xauth add Ubuntu/unix:0  MIT-MAGIC-COOKIE-1  23aa75decfb3078befff7f57f6350f59
```

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241121153614.png)

### 5.2 先启动系统再插入屏幕无显示问题

Xorg在系统启动的时候若没有检测到屏幕，会初始化一个1024x768的framebuffer，之前遇到在使用lightdm和lightdm-gtk-greeter的这种桌面环境，在先启动系统，后接入屏幕的这种情况下，显示屏无法正常显示，切换成ukui-greeter可以显示。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241121175136.png)

#### 5.2.1 问题分析

1. 通过Xorg的log可以看出显示器的显示模式获取是正常的，说明和屏幕的连接是通的，显示器的edid已经读到了
![](https://raw.githubusercontent.com/JackHuang021/images/master/20241121175542.png)

2. xrandr查看屏幕显示模式是否设置正确，发现屏幕显示模式设置有问题，从上面的信息可以看到DP-1的屏幕虽然已经连接，但是分辨率是没有设置的，正常要是设置了分辨率，对应的分辨率后面会显示+号，通过手动设置屏幕分辨率后`xrandr --output DP-1 --mode 1280x720`，屏幕显示正常，说明是插入屏幕后Xorg对屏幕分辨率的设置有问题：

   ```bash
   jack@Ubuntu:~$ xrandr
   Screen 0: minimum 320 x 200, current 1024 x 768, maximum 16384 x 16384
   DP-1 connected primary (normal left inverted right x axis y axis)
      1280x720      60.00    50.00    59.94  
      1024x768      70.07    60.00  
      832x624       74.55  
      800x600       72.19    75.00    60.32    56.25  
      720x576       50.00  
      720x480       60.00    59.94  
      640x480       75.00    72.81    66.67    60.00    59.94  
   DP-2 disconnected (normal left inverted right x axis y axis)
   ```

3. 通过Xorg插入屏幕后的log，使用gdb调试在相应位置打断点，查看Xorg的执行流程，下面是插入显示器后的Xorg堆栈，查看Xorg的源码可以分析插入屏幕后的执行流程为：`RRGetInfo() -> RRTellChanged() -> TellChanged() -> RRDeliverScreenEvent(client, pWin, pScreen);`，可以看到Xorg只是发出了一个RRScreenChangeNotify的通知到X11客户端，并未直接处理屏幕分辨率的设置
   ![](https://raw.githubusercontent.com/JackHuang021/images/master/20241119111333.png)

   ![](https://raw.githubusercontent.com/JackHuang021/images/master/20241121180552.png)

4. 分析ukui-greeter为什么能显示桌面，但是屏幕显示的分辨率为1024x768，显示分辨率不对。通过分析ukui-greeter的源码，发现其响应了Xorg发出的ScreenChangeEvent事件，然后通过调用xrandr命令进行了屏幕分辨率的设置。代码执行流程为`RRScreenChangeEvent() -> onScreenCountChanged() ->  QString  strXrandr = displayService.getFirstDisplayXrandrCmd(); -> enableMonitors.start(strXrandr);`，调用的xrandr命令为`xrandr --output " + monitorNames[0] + " --auto`
   ![](https://raw.githubusercontent.com/JackHuang021/images/master/20241121180802.png)
   ![](https://raw.githubusercontent.com/JackHuang021/images/master/20241121181037.png)

**总结**：所以在使用lightdm-gtk-greeter的情况下，可能还需要对其代码进一步修改，需要相应Xorg的ScreenChangeEvent事件，然后设置屏幕的分辨率。另外一种解决办法，也可以通过udev规则，在插入屏幕的时候执行xrandr，将屏幕分辨率设置为最佳分辨率（之前在gitee wiki给出的脚本需要重新启动lightdm，这样会导致插拔屏幕会进入到登录页面，使用起来体验不好）。

#### 5.2.2 显示器热插拔udev规则

1. 编写脚本： /usr/local/bin/auto_resolution.sh

   ```bash
   #!/bin/bash

   # 处理X11的认证问题
   export DISPLAY=:0
   export XAUTHORITY=$HOME/.Xauthority
   xauth add $(xauth -f /var/run/lightdm/root/:0 list)

   # 查看当前屏幕的连接情况
   connected_output=$(xrandr --query | grep " connected" | awk '{print $1}')
   for output in $connected_output; do
      # 设置最佳分辨率
      preferred_mode=$(xrandr --query | grep -A 1 "^$output" | tail -n 1 | awk '{print $1}')
      if [ -n "$preferred_mode" ]; then
         echo "Setting $output to $preferred_mode"
         xrandr --output "$output" --mode "$preferred_mode"
      fi
   done
   ```

2. 编写udev规则： /etc/udev/rules.d/99-monitor-hotplug.rules

   ```bash
   SUBSYSTEM=="drm", ACTION=="change", RUN+="/usr/local/bin/auto_resolution.sh"
   ```

3. 加载udev规则

   ```bash
   udevadm control --reload-rules
   ```
