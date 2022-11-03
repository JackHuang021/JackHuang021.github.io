---
title: Ubuntu编译QFtp并使用
author: Jack
tags:
  - Linux
  - QFtp
categories:
  - Linux
abbrlink: 1b5a9595
date: 2022-06-27 14:18:15
---

#### 下载QFtp源码
[QFtp源码](https://github.com/qt/qtftp.git)  
`gti clone https://github.com/qt/qtftp.git`

#### 编译QFtp模块
在QtCreator上编译出了点问题，只能在终端进行编译
+ 进入源码目录`cd src/qftp`，修改pro文件`qftp.pro`,修改如下
<!-- more -->
```
load(qt_build_config)

TARGET = QtFtp
CONFIG += static
CONFIG -= shared
QT = core network

MODULE_PRI = ../../modules/qt_ftp.pri
MODULE = ftp

load(qt_module)

# Input
HEADERS += qftp.h qurlinfo.h
SOURCES += qftp.cpp qurlinfo.cpp

```
修改`qurlinfo.cpp`中的`qurlinfo.h`路径，修改如下
```'
#include "qurlinfo.h"

#include "qurl.h"
#include "qdir.h"
#include <limits.h>

QT_BEGIN_NAMESPACE
```
+ 在终端中进入源码目录`cd src/qftp`，运行`qmake`，之后会生成Makefile
+ `make`生成`libQt5Ftp.a`静态库，pri模块文件
+ `make install`将生成的库文件及QFtp头文件复制到Qt安装目录
+ 对于交叉编译环境下其他平台的编译也可按照上面的步骤，qmake需要替换交叉编译环境下对应的qmake

#### QFtp使用
+ 官方源码目录example文件夹下有一个例程，网上有大佬稍加修改上传到了GitHub，[QFtp例程](https://github.com/chuanstudyup/QFtpExample.git)  
+ 下载这个例程，上述编译步骤没问题的话，直接编译运行即可  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220627150335.png)