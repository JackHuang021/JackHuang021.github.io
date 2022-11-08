---
title: ssh使用密钥实现免密登录
author: Jack
tags:
  - Linux
  - ssh
categories:
  - Linux
abbrlink: 37d9659b
date: 2022-06-01 08:38:13
---

### 本地ssh客户端准备ssh密钥
ssh密钥默认保存路径`~/.ssh`，进入该目录查看是否已存在生成的密钥
```
ls -al ~/.ssh
```
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220601093641.png)
<!-- more -->
如上图，`id_rsa.pub`和`qtc_id.pub`都是公钥  
如果没有公钥，可以使用`ssh-keygen`生成

### 上传ssh公钥到ssh服务器
```
ssh-copy-id -i ~/.ssh/id_rsa.pub username@ip_address
```
执行后会提示输入服务器用户密码

### 测试ssh免密登录
```
ssh username@ip_address
```
如果可以直接登录，说明已经配置成功  
都2022年了，不要在输入密码上再浪费更多时间了

### 删除免密登录
上传公钥后，服务端`.ssh/authorized_keys`文件中会添加一行内容，就是本地客户端的公钥，编辑该文件删除改行，即可禁用客户端免密登录  
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220601094901.png)