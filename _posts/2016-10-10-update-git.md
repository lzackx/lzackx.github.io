---
layout: post
title:  "Mac上更新git"
date:   2016-10-10 09:00:00
categories: tools
---
**Mac上更新git步骤**

**安装[Homebrew](http://brew.sh/)**

Homebrew是Mac上非常好用套件管理器，如果嫌自己编译获取新版本git麻烦，使用`brew`命令安装需要的命令行工具是个不错的选择。
具体Homebrew如何安装，直接点击上面的链接进入官网即可知道，支持多语言，什么都不懂也能快速搞定。

**更新[git](https://github.com/git/git)**

git这轻量的版本管理工具，基本开发者都知道。
但由于在Mac上安装Xcode顺带安装的git版本并不是很跟得上开源社区的进度，所以需要的时候，是有必要手动更新自己的git版本的。

查看git版本号:

`git --version`

查看git的路径:

`which git`

貌似`brew`说2016后不允许用`sudo`来执行，那么`brew`不能升权限执行，那就把需要执行的位置降低权限`sudo chown -R $USER:admin /usr/local`，

卸载git:

`brew uninstall git`

安装git:

`brew install git`

更新的时候，按照习惯，先卸载了，再安装,也是很简单。
