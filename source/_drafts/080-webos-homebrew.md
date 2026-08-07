---
title: WebOS 破解教程
tags:
---


**风险提示：** 本教程仅供技术研究和学习使用，Root 电视存在风险，可能导致设备无法正常使用。请务必严格按照步骤操作。

## 1. 前期准备

工具准备：

+ 一台可以联网的电脑
+ 网络连接（确保电视与电脑处于同一局域网）
+ 安装 SSH/Telnet 客户端工具（推荐使用 OpenSSH），OpenSSH 下载地址：[https://www.mls-software.com/files/setupssh-9.9p1-1.exe](https://www.mls-software.com/files/setupssh-9.9p1-1.exe)，下载后打开点下一步进行安装。
+ 设备型号和软件版本确认： 在 系统设置->常规设置->关于电视 查看，记录 “软件版本” 和 “型号”
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250401110905.png)

## 2. 查询是否能 root

在 PC 浏览器上访问 [https://cani.rootmy.tv/](https://cani.rootmy.tv/)，输入电视型号和软件版本，例如 `OLED65G2PUA 04.40.75`（注意型号和软件版本中间有个空格）, faultmanager 显示深绿色表示电视可以root

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250401110622.png)

## 3. 开启开发者模式

1. 注册 LG WebOS 开发者账号，访问网址[https://webostv.developer.lge.com/](https://webostv.developer.lge.com/)，右上角点击 `SIGN IN`进行账号注册
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250401102734.png)
2. 打开 LG Content Store： 在电视上找到并进入 LG Content Store。
3. 搜索 Developer Mode： 找到“开发者模式”应用并安装。
4. 打开 “开发者模式”应用，根据提示登录刚刚注册的 LG 开发者账号
5. 启用调试模式： 在 Developer Mode 中，启用调试选项（Dev Mode Status），启用Key Server；并记录下电视显示的 IP 地址，6位数的密钥，根据提示重启电视

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250401104557.png)

## 4. Root 电视

1. 确保同一局域网： 确保电视和电脑处于同一 WiFi 网络或通过网线连接至同一路由器。
2. 下载电视密钥文件：在电脑浏览器访问 `http://<TV_IP>:9991/webos_rsa` ， 例如你的电视IP地址为 `192.168.1.2`，则访问 `http://192.168.1.2:9991/webos_rsa`，访问后会弹出下载框，将文件保存到电脑，命名为 `webos_rsa`
3. ssh 连接电视：打开命令行工具，输入`ssh -i webos_rsa -p 9922 prisoner@<TV_IP>`，注意 `webos_rsa` 文件路径和电视的IP地址，会提示输入密码，输入之前记录的6位数密钥（区分大小写），连接成功后不会有其它提示，可以输入`ls`再回车，看看是否有信息打印出来
4. 开始 Root：复制下面的命令到命令行窗口，回车运行，等到出现 “Payload complete” 的打印，表示 root 成功了。

	```bash
	curl -L -o /tmp/autoroot.sh -- 'https://raw.githubusercontent.com/throwaway96/faultmanager-autoroot/refs/heads/main/autoroot.sh' &&
	sh /tmp/autoroot.sh
	```

5. **卸载掉 `LG Developer Mode` 应用，必须要在重启前进行卸载**
6. 重启电视

## 5. 安装软件

电视 root 后，会自动安装 `Homebrew Channel` 这个软件，打开`Homebrew Channel`，安装需要的软件（注意安装无广告的油管前必须先卸载已经在应用市场安装的油管）
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250401111519.png)
