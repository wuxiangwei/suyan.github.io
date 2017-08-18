---
layout: post
title: Git Submodule（子模组）
category: [工具]
tags: [Git]
keywords: Git
---

```
hexo-site    --- hexo-site版本库
└── themes
    └─nova --- nova版本库
```

需求：一个版本库引用其它版本库的文件。如上所示，hexo-site版本库引用nova版本库中的文件。


## 基本操作

### 添加Submodule

```
cd hexo-site/
git submodule add -f https://github.com/wuxiangwei/hexo-theme-nova.git themes/nova
```
执行添加子模块命令后，hexo-site目录中将生成1个新文件**.gitmodule**，文件记录submodule的引用信息，包括在当前项目的位置以及仓库所在。

### 查看Submodule

```
git submodule status 
dabe550a44b98194546515ec78f8ec8c11b41340 themes/nova (v0.1.2-13-gdabe550)
```
可以看到每个Submodule的commit id，这commit id是固定的，每次克隆主版本库时不变，外部版本库更新时不变。

### 克隆带Submodule的版本库

克隆带Submodule的版本库，并不能自动克隆Submodule的版本库。这个特点的好处是，克隆版本库速度快，数据没有冗余。

```
git submodule init
git submodule update
```
使用上述两个命令能够克隆Submodule的版本库。

### 同步Submodule版本库的修改

使用场景：Submodule版本库的主线(hexo-theme-nova)已修改，但这修改没有同步到主版本库(hexo-site)。

Submodule版本库的主线的commit：
```
root@bs-dev:~/repo/hexo-theme-nova# git log --pretty=oneline
6dccc29d93b0928a402290a3d9a0d2b1fd57e031 关闭打赏页面
dabe550a44b98194546515ec78f8ec8c11b41340 Merge pull request #15 from Jamling/dev
```

主版本库的Submodule版本的commit：
```
root@bs-dev:~/repo/hexo-site# git submodule status 
-dabe550a44b98194546515ec78f8ec8c11b41340 themes/nova
```

克隆Submodule版本库时(执行git submodule update命令)，自动将Submodule版本库checkout到给定的commit，进入Submodule路径可以查看到版本库的状态。在主版本中同步Submodule修改的步骤如下:

```
cd hexo-site/themes/nova
git checkout master

cd hexo-site
git status
```
进入Submodule版本库/hexo-site/themes/nova，checkout出master分支
进入主版本库/hexo-site，发现theme/nova已修改，提交修改。


## .gitmodule文件

```
[submodule "themes/nova"]
	path = themes/nova
	url = https://github.com/wuxiangwei/hexo-theme-nova.git
```
添加Submodule后，将在hexo-site目录中生成.gitmodule文件用以记录Submodule的引用信息，包括在主版本库中的路径以及仓库地址。


