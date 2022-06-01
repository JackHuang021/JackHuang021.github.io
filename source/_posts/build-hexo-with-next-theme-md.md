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
安装过程中可能会因为网路问题失败，可以多重试几次或挂梯子
```
npm install -g hexo-cli
```

### 安装Next主题
Next主题是Hexo比较知名的第三方主题，极简风格，有相当多的使用者，维护也做得比较好  
不过Next新旧版本的仓库地址不一样，目前最新的GitHub地址[hexo-theme-next](https://github.com/next-theme/hexo-theme-next.git)  
Next主题安装比较简单，直接从仓库clone然后修改Hexo配置文件
```
cd blog
git clone https://github.com/next-theme/hexo-theme-next themes/next
```
修改Hexo配置文件`_config.yml`，将站点主题改为Next，修改如下
```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next
```

全部安装完成后的版本信息如下
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
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/imageshexo_next_theme.png)

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
后续更新博客设置或者文章的话需要再次进行上传部署`hexo g -d`

### Hexo博客备份
Hexo在进行部署时，是将页面内容解析后放在`.depoly_git`中进行上传GitHub仓库，博客内文章源文件并未进行上传，所以还需要手动将这些文件进行手动上传。目前比较常用的方法是在原GitHub仓库建立一条分支，将这些文件上传到该分支。
```
cd blog 
git init 
git submodule add https://github.com/next-theme/hexo-theme-next.git themes/next
git add .
git commit -m "init blog backup"
git branch -m master hexo
git remote add origin git@github.com:JackHuang021/JackHuang021.github.io.git
git push -u origin hexo
```

### 恢复Hexo博客
1. 按照之前的步骤搭建Hexo环境
2. clone之前备份的hexo分支内容`git clone --recursive -b hexo git@github.com:JackHuang021/JackHuang021.github.io.git blog`
3. 下载npm依赖模块
```
cd blog
npm install
```
4. clone master分支内容
```
cd blog
git clone git@github.com:JackHuang021/JackHuang021.github.io.git .deploy_git 
```
5. 正常更新、部署博客
```
hexo g
hexo s
hexo d
```

### 关于Hexo使用的思考
我觉得Hexo最大的特点就是便捷，借助GitHub可以在多台设备中无缝切换进行博客写作，服务器的维护工作基本不需要作者进行，换设备后直接搭建hexo环境，从GitHub拉取博客内容即可。
