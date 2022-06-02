---
title: imx6ull OpenCV4.5 交叉编译
author: Jack
tags:
  - imx6ull OpenCV 交叉编译
categories:
  - OpenCV
abbrlink: e4359116
date: 2022-06-02 09:44:44
---

#### 交叉编译环境
+ Ubuntu版本：Ubuntu20.04 64bits
+ 交叉编译工具：arm-linux-gnueabihf-
+ 硬件平台正点原子IMX6ULL (ALPHA)

<!-- more -->

#### 准备源码和交叉编译工具链
Linux环境下的编译方法可以参考[Opencv安装官网教程](https://docs.opencv.org/4.x/d7/d9f/tutorial_linux_install.html)
```
wget -O opencv.zip https://github.com/opencv/opencv/archive/4.x.zip
unzip opencv.zip
```
交叉编译工具链版本  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602095241.png)

#### 安装cmake和cmake-gui工具
在命令行使用cmake工具确实很不方便，cmake-gui配置起来比较省时间  
`sudo apt-get install cmake cmake-qt-gui  cmake-curses-gui`  

#### 配置交叉编译环境 
1. 运行cmake-gui
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602100408.png)  
2. 在第一个框输入OpenCV源码路径，在第二个框输入OpenCV编译目录  
3. 点击`Configure`配置交叉编译环境  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602100750.png)
4. 选择`Spcify options for cross-compile`  
5. 按照下图设置交叉编译工具链  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602101028.png)
6. 点击`Finish`回到cmake-gui主页面，勾选Advanced  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602101237.png)


#### 配置cmake选项
1. 在CMAKE_EXE_LINKER_FLAGS处添加上`-lpthread -lrt -ldl`
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602101546.png)
2. 在CMAKE_INSTALL_PREFIX处指定安装目录，如果不指定，它会默认安装到Ubuntu系统目录`/usr/local`下。  
3. 取消`BUILD_opencv_gapi`选项，不取消这个选项后续编译的时候会报错
4. 再依次点击`Configure`， `Generate`，击了Generate后看到信息像如下图一样，表明生成成功，一般按照上面配置后基本都不会报错。  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220602102258.png)

#### 编译OpenCV
1. 打开之前设置的imx6编译目录，该目录下有刚才生成的Makefile  
2. 输入`make -j12`开始编译
3. 编译完成后输入`make install`，OpenCV的库和头文件会安装到之前设置的`CMAKE_INSTALL_PREFIX`目录

#### 编译过程中遇到的错误
```
/home/tanyd/zdyz/linaro494/arm-linux-gnueabihf/libc/usr/include/features.h:311:52: error: operator '&&' has no right operand #if defined _FILE_OFFSET_BITS && _FILE_OFFSET_BITS == 64
```
解决方法： 在`#if defined`前面加上 `#define _FILE_OFFSET_BITS 64`  