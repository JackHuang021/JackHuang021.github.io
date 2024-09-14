---
title: GL-MT1300安装ssr plus
tags:
  - openwrt
categories:
  - daily
abbrlink: 4b85cd46
date: 2024-09-14 16:41:56
---

GL-MT1300这款路由器目前在咸鱼上的价格还可以，有三个千兆口，颜值很高，双频wifi，不过处理器差一点，flash只有32MB。正好办公的网络环境比较复杂，买了一个来改善办公网络环境。

在编译安装ssr plus的过程中，遇到了一些问题，记录一下解决过程。

<!-- more -->
## 1. 使用sd卡作为根文件系统
MT1300默认可用的flash空间大约是15MB，由于编出来的v2ray-core的安装包就有8MB，不使用sd卡挂载来做文件系统的话，装不下完整的ssr plus插件（不使用sd卡只能装上ss，仅使用ss协议可以不需要sd卡来做文件系统）
```bash
# 安装必要工具
root@GL-MT1300:/# opkg update
root@GL-MT1300:/# opkg install block-mount  kmod-usb-storage  kmod-fs-ext4 e2fsprogs
# 格式化SD卡，我这里SD卡的设备节点为/dev/mmcblk0p1：
root@GL-MT1300:/# mkfs.ext4 /dev/mmcblk0p1

# 制作根文件系统
root@GL-MT1300:/# mount /dev/mmcblk0p1 /mnt
root@GL-MT1300:/# mkdir /tmp/root
root@GL-MT1300:/# mount -o bind / /tmp/root
root@GL-MT1300:/# cp /tmp/root/* /mnt -a
root@GL-MT1300:/# umount /tmp/root
root@GL-MT1300:/# umount /mnt

# 配置自动挂载并重启路由器
root@GL-MT1300:/# block detect > /etc/config/fstab
root@GL-MT1300:/# uci set fstab.@mount[0].target='/overlay'
root@GL-MT1300:/# uci set fstab.@mount[0].enabled='1'
root@GL-MT1300:/# uci commit fstab
root@GL-MT1300:/# reboot
```

> 我这里使用v4.3.19的固件会出现wan口dhcp获取不到IP的情况，后来刷回到v4.3.18固件正常了

## 2. 编译ssr plus
### 2.1 准备openwrt源码
1. 先在 系统->概要 里面查看openwrt的版本是多少，后续我们需要使用对应版本的openwrt来编译插件，例如我这台的版本是openwrt 22.03.4

![](https://raw.githubusercontent.com/JackHuang021/images/master/20240914164856.png)

2. 下载openwrt的源码，并切到对应版本的tag点
```bash
git clone https://github.com/openwrt/openwrt
cd openwrt
git checkout v22.03.4
```

3. 修改feeds.conf.default，加入ssr plus的组件包
```diff
--- a/feeds.conf.default
+++ b/feeds.conf.default
@@ -1,4 +1,6 @@
-src-git-full packages https://git.openwrt.org/feed/packages.git^38cb0129739bc71e0bb5a25ef1f6db70b7add04b
+#src-git-full packages https://git.openwrt.org/feed/packages.git^38cb0129739bc71e0bb5a25ef1f6db70b7add04b
+src-git packages https://github.com/coolsnowwolf/packages
 src-git-full luci https://git.openwrt.org/project/luci.git^ce20b4a6e0c86313c0c6e9c89eedf8f033f5e637
 src-git-full routing https://git.openwrt.org/feed/routing.git^1cc7676b9f32acc30ec47f15fcb70380d5d6ef01
 src-git-full telephony https://git.openwrt.org/feed/telephony.git^5087c7ecbc4f4e3227bd16c6f4d1efb0d3edf460
+src-git helloworld https://github.com/fw876/helloworld.git
```

### 2.2 编译openwrt
1. 修改编译选项
```bash
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig -j12
```

选择我们需要编译的设备
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240914170319.png)

选择需要编译的插件，按照下面图片中的选上
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240914171824.png)

2. 开始编译`make V=s -j12`，一般都能编译通过，编译后的安装包路径为`bin/packages/mipsel_24kc`，ssr plus的安装包在`helloworld`目录中，可以把这个目录下的所有包都拷贝到路由器上进行安装，并留一个备份

## 3. 安装ssr plus
1. ssr plus的安装需要其它几个依赖的软件包，我们先安装其它的包，最后再装ssr plus
```bash
opkg install chinadns-ng_2023.10.28-1_mipsel_24kc.ipk
opkg install dns2socks_2.1-2_mipsel_24kc.ipk
opkg install dns2tcp_1.1.2-1_mipsel_24kc.ipk
opkg install lua-neturl_1.1-1-3_all.ipk
opkg install microsocks_1.0.4-1_mipsel_24kc.ipk
opkg install shadowsocksr-libev-ssr-check_2.5.6-11_mipsel_24kc.ipk
opkg install shadowsocksr-libev-ssr-local_2.5.6-11_mipsel_24kc.ipk
opkg install shadowsocksr-libev-ssr-redir_2.5.6-11_mipsel_24kc.ipk
opkg install simple-obfs-client_0.0.5-1_mipsel_24kc.ipk
opkg install tcping_0.3-1_mipsel_24kc.ipk
opkg install v2ray-core_5.16.1-1_mipsel_24kc.ipk
opkg install v2ray-plugin_5.15.1-1_mipsel_24kc.ipk

opkg install luci-app-ssr-plus_188_all.ipk --force-overwrite
```

2. 安装完之后可能还会遇到打不开ssr插件和订阅无法更新的问题，还需要安装下面几个包：
```bash
opkg install luci-compat
opkg install wget-ssl
opkg install luci-lib-ipkg
```
安装完这些包后应该是可以正常使用了
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240914173249.png)

