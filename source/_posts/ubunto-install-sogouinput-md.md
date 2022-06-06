---
title: Ubuntu安装搜狗输入法的问题记录
author: Jack
tags:
  - Linux
  - 搜狗输入法
categories:
  - Linux
abbrlink: 245a6c83
date: 2022-06-06 09:15:55
---

+ 最近在帮一个客户安装搜狗输入法时遇到了安装后无候选框的问题，记录一下该问题的解决办法，今天进入官网查看，官网已经修改安装指南，建议按照官网指南安装，本文截图来自搜狗官网

#### Ubuntu搜狗输入法下载
+ [搜狗输入法Linux版官网](https://pinyin.sogou.com/linux?r=pinyin)，目前最新版本为V4.0.1
+ [搜狗输入法Linux版安装指南](https://pinyin.sogou.com/linux/guide)

#### Ubuntu20.04安装步骤
1. 添加中文语言支持，打开 系统设置——区域和语言——管理已安装的语言——在“语言”tab下——点击“添加或删除语言”
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220606092214.png)
2. 弹出“已安装语言”窗口，勾选中文（简体），点击应用
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220606092221.png)
3. 安装fcitx输入法框架`sudo apt-get install fcitx`
4. 卸载系统ibus输入法框架`sudo apt purge ibus`
5. 回到“语言支持”窗口，在键盘输入法系统中，选择`fcitx`
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220606092226.png)
6. 通过命令行安装搜狗输入法`sudo dpkg -i sogoupinyin_版本号_amd64.deb`，如果安装过程中提示缺少相关依赖，则执行如下命令解决：`sudo apt -f install`
7. 安装输入法依赖
```
sudo apt install libqt5qml5 libqt5quick5 libqt5quickwidgets5 qml-module-qtquick2
sudo apt install libgsettings-qt1
```
8. 点击“应用到整个系统”，关闭窗口，重启电脑
9. 查看状态栏右上角，可以看到“搜狗”字样，在输入窗口即可使用搜狗输入法。没有“搜狗”字样，选择配置，将搜狗加入输入法列表即可。  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220606094224.png)
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220606094237.png)
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220606094048.png)

#### 问题记录
+ 安装完成后输入候选框不出现，只能输入英文
+ 解决办法，安装qt依赖库可以解决
```
sudo apt install libqt5qml5 libqt5quick5 libqt5quickwidgets5 qml-module-qtquick2
sudo apt install libgsettings-qt1
```
