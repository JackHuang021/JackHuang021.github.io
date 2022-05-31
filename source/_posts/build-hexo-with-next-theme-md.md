---
title: 从零开始搭建Hexo博客
author: Jack
top: true
abbrlink: 39fb7b7f
date: 2022-05-31 18:07:21
tags:
toc: true
---

## Hexo简介
引用官网的介绍：A fast, simple & powerful blog framework  
Hexo是一款基于Node.js的静态博客框架，依赖少易于安装使用，可以方便的生成静态网页托管在GitHub上，是搭建博客的首选框架。

## 环境搭建
基于Ubuntu20.04安装Hexo配置Next主题
<!-- more -->
### 安装Git
```
sudo apt-get install git
```
安装完成后进行配置
```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```
### 安装Node.js
使用NVM（Node Version Manager）方式进行安装
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.2/install.sh | bash
```
安装完成后关闭终端重新打开

### 安装最新版本node
安装过程中可能会因为网络问题失败，可以尝试挂梯子,安装完成后更新自带npm
```
nvm install node
npm install npm -g
```

### 安装Hexo
```
npm install -g hexo-cli
```
安装完成后的版本信息如下
```

jack@linux:~/blog/source/_posts$ hexo -v
INFO  Validating config
INFO  ==================================
  ███╗   ██╗███████╗██╗  ██╗████████╗
  ████╗  ██║██╔════╝╚██╗██╔╝╚══██╔══╝
  ██╔██╗ ██║█████╗   ╚███╔╝    ██║
  ██║╚██╗██║██╔══╝   ██╔██╗    ██║
  ██║ ╚████║███████╗██╔╝ ██╗   ██║
  ╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝   ╚═╝
========================================
NexT version 8.11.1
Documentation: https://theme-next.js.org
========================================
hexo: 5.4.2
hexo-cli: 4.3.0
os: linux 5.13.0-40-generic Ubuntu 20.04.4 LTS (Focal Fossa)
node: 18.2.0
v8: 10.1.124.8-node.13
uv: 1.43.0
zlib: 1.2.11
brotli: 1.0.9
ares: 1.18.1
modules: 108
nghttp2: 1.47.0
napi: 8
llhttp: 6.0.6
openssl: 3.0.3+quic
cldr: 41.0
icu: 71.1
tz: 2022a
unicode: 14.0
ngtcp2: 0.1.0-DEV
nghttp3: 0.1.0-DEV
```

### 初步运行Hexo进行验证
```
hexo init blog
cd blog
npm install
hexo server
```
运行之后可以通过 http://localhost:4000 进行访问
```
INFO  Validating config
INFO  ==================================
  ███╗   ██╗███████╗██╗  ██╗████████╗
  ████╗  ██║██╔════╝╚██╗██╔╝╚══██╔══╝
  ██╔██╗ ██║█████╗   ╚███╔╝    ██║
  ██║╚██╗██║██╔══╝   ██╔██╗    ██║
  ██║ ╚████║███████╗██╔╝ ██╗   ██║
  ╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝   ╚═╝
========================================
NexT version 8.11.1
Documentation: https://theme-next.js.org
========================================
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
### 部署GitHub远程服务器
#### 创建GitHub项目
在GitHub上注册账号，注册后上传ssh公钥，便于后续的部署操作  
创建一个与你用户名对应的项目`username.github.io`，例如我创建的项目地址为`https://github.com/JackHuang021/JackHuang021.github.io.git`  
#### 进行部署
部署需要用到`hexo deploy`上传到GitHub仓库，这里需要下载部署插件，并修改hexo配置文件`_config.yml`，
我们很多的博客设置都可以在这个配置文件里面进行修改  
首先安装Git部署插件
```
cd ~/blog
npm install hex-deployer-git --save
```
修改博客配置文件`_config.yml`，增加如下内容
```
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: git@github.com:JackHuang021/JackHuang021.github.io.git
  branch: master
```
最后使用`hexo d`进行上传部署，现在访问`username.github.io`便可以看到博客页面了  
后续更新博客设置或者文章的话需要再次进行上传部署`hexo d -g`
