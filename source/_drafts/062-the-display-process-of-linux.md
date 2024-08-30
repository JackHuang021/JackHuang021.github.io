---
title: Linux 桌面显示流程（X11协议）
date: 2024-01-02 15:35:35
tags:
---

Linux图像显示采用的是X Window System（X, X11）这套协议，我们平常经常见到的X就是指的这套协议，这套协里面规定的架构是C/S架构，即客户端/服务端架构，客户端是x client，服务端是x server


服务端（x server）负责管理键盘、鼠标、显卡、显示器等输入输出设备；客户端收到来自键盘、鼠标等输入设备的数据，通知服务端进行显示，服务端通过控制输出设备进行输出。客户端和服务端可以不用在一台设备上，它们可以通过网络进行通讯

客户端（x client）就是计算机上运行的程序（比如浏览器等），客户端想要显示图形，必须通过服务端，由服务端来负责显示

### 显示管理器（Display Manager）
显示管理器，也被叫做登录管理器，systemd负责启动显示管理器，显示管理器再去启动xserver，并显示一个登录界面，输入用户名和密码后，显示管理器通过PAM模块完成用户认证，认证成功的话再去启动窗口管理器，比较常见的显示管理器有GDM、KDM、LightDM

#### systemd unit files目录
一共有三个目录定义了unit文件，分别是`/lib/systemd/system`、`/etc/systemd/system`、`/run/systemd/system/`，他们的优先级是`/etc > /run > /lib`

#### lightdm
seat的概念：一套显示及输入输出设备，Multi-seat是更进一步的概念，它允许多个用户同时使用，假如有两个seat，在登录界面就会有两个greeter让这两个人分别输入用户名和密码，登录到他们各自的桌面环境。

##### lightdm代码分析
lightdm repository: [https://github.com/canonical/lightdm.git](https://github.com/canonical/lightdm.git)

在`src/lightdm.c`中，`/var/run/lightdm.pid`保证单实例运行，加载配置文件

创建display_manager，关联信号`DISPLAY_MANAGER_SIGNAL_STOPPED`，`DISPLAY_MANAGER_SIGNAL_SEAT_REMOVED`
```c
display_manager = display_manager_new ();
g_signal_connect (display_manager, DISPLAY_MANAGER_SIGNAL_STOPPED, G_CALLBACK (display_manager_stopped_cb), NULL);
g_signal_connect (display_manager, DISPLAY_MANAGER_SIGNAL_SEAT_REMOVED, G_CALLBACK (display_manager_seat_removed_cb), NULL);
```

