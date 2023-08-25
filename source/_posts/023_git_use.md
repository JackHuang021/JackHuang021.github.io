---
title: git使用记录
tags: git
categories: Code
abbrlink: 21d76c15
date: 2022-11-17 16:14:27
---

本文记录git使用过程中用到的一些功能
<!-- more -->

#### 创建worktree
在各个差异较大的分支中切换，工程需要做全局编译，导致开发效率下降，创建worktree可以在多个分支并行开发，从而实多个工程环境的缓存，达到提升开发效率的目的，特点：
+ 可为一个分支创建一个工作区
+ 每个工作区的工程环境独立运行
+ 每个工作区共享同一个版本仓库信息

相比通过git clone方式创建多个独立工程环境的工作区，git worktree的优点在于：
+ **更节省硬盘空间**：`git clone`方式下，每个工作区都有一个版本仓库，`git worktree`方式下，每个工作区共享同一个版本仓库，节省了n-1/n（n为工作区数量）的硬盘空间
+ **各个工作区之间的更新同步更快**：`git clone`方式下，A工作区和B工作区同步更新的步骤，A工作区commit -> A工作区push -> B工作区pull，`git worktree`方式下，A工作区只要本地提交更新后，其他工作区就能立即收到

**git worktree使用**
```bash
# 进入工程目录
cd path/to/project
# 创建worktree
git worktree add worktree test_branch
# 进入工作区worktree上进行开发
cd worktree

# 删除工作区
rm -rf worktree

# 清理工作区信息
git worktree prune
```

#### 合并其他分支commit到当前分支
使用`git cherry-pick`将其他分支的commit合并到另一个分支
```bash
git cherry-pick <commit-id>
```
同时也支持批量合并，一次可以cherry-pick一个区间内的commit
```bash
git cherry-pick <start-commit-id>..<end-commit-id>
```

#### git生成patch
使用`git format-patch`生成patch
```bash
# 生成单个commit的patch
git format-patch -1 <commit id>

# 生成最近1次commit的patch
git format-patch HEAD^

# 生成最近2次commit的patch
git format-patch HEAD^^

# 生成两个commit间的修改的patch
# 生成的patch不包含commit id1
git format-patch <commit id1>..<commit id2>
```
合入patch
```bash
# 检查patch是否能正常合入
git apply --check <path/to/patch>

# 合入patch
git apply <path/to/patch>
```

#### git PR过程
1. 在Git主页fork开源项目，然后将fork后的项目仓库clone到本地
```bash
git clone git@gitlab.phytium.com.cn:huangjie1663/linux-kernel.git
```
2. 添加远程代码仓库地址
```bash
jack@linux: git remote add upstream git@gitlab.phytium.com.cn:embedded/linux/linux-kernel.git
jack@linux: git remote -v
origin	git@gitlab.phytium.com.cn:huangjie1663/linux-kernel.git (fetch)
origin	git@gitlab.phytium.com.cn:huangjie1663/linux-kernel.git (push)
upstream	git@gitlab.phytium.com.cn:embedded/linux/linux-kernel.git (fetch)
upstream	git@gitlab.phytium.com.cn:embedded/linux/linux-kernel.git (push)
```
这时总共包含三个仓库信息：本地仓库、origin仓库、upstream仓库

3. 更新git remote中所有远程仓库所包含分支的最新commit，并合并upstream
```bash
git fetch upstream
git merge upstream/branch
```

#### git代码回滚
git代码回滚常用的两种方式：`git revert`和`git reset`

git reset，将提交的commit从历史记录中删除，用途如下：
+ 新提交的commit有问题，想要撤销该笔提交，并保留修改的内容`git reset --soft HEAD^`
+ 撤销新提交的commit，不保留修改的内容`git reset --hard HEAD^`

#### git暂存文件转为unstage
```bash
# 当前所有暂存文件全部转为unstage状态
git restore --staged .

# 将某个暂存文件转为unstage状态
git restore --staged <filename>
```

