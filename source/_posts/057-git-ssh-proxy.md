---
title: git ssh代理设置
date: 2023-10-07 15:30:34
tags:
  - git
  - ssh
  - proxy
categories: Code
---

最近发现在开启了代理后，使用git push, git fetch时还是偶尔会等待很久，有时候急需上传和拉取代码时卡住很是头痛
<!-- more -->

我一般使用ssh协议连接远程仓库，在ssh config（~/.ssh/config）中为github.com设置代理后问题解决了，记录一下

在`~/.ssh/config`中增加下面的内容
```bash
Host github.com
	HostName github.com
	User git
	ProxyCommand nc -v -x 127.0.0.1:1089 %h %p
# 根据代理的端口修改127.0.0.1:1089
```
