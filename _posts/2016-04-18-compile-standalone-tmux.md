---
layout:	post
title:	静态编译tmux
date:	Mon, 18 Apr 2016
categories:	效率工具
tags:	tmux 编译
---

办公环境中ssh登录的Linux服务器版本太旧了，很多库和软件都已经是很早的版本了，而且就算是这些老旧的软件还没有权限安装任何东西。看了看服务器版本的软件源中的tmux还是1.3版本的，在debian的package中找了找，现在都找不到提供这个版本的tmux deb包了，虽然最后在网上找到了tmux 1.3的deb包，又找到libevent1.4的包都放在我的`$HOME/bin`下，折腾挺久的终于启动了tmux，却发现我的`tmux.conf`中一些配置和命令在这个版本中不支持。想用到最新的tmux，那依赖的库也要升级，最麻烦的是试了试升级libc6后我的其他命令（如ls）都不能正常用了，于是打算把新版tmux编译成独立的可执行文件，把依赖库都静态编译进去，虽然最终文件会大点，但是对于办公环境的服务器可以不升级任何库就使用。

## 源码

tmux最新版已经到2.2了，但是我笔记本的openSUSE源上现在只有2.0的，就先和我笔记本上的tmux版本保持一致。

从[tmux release](https://github.com/tmux/tmux/releases)下载tmux 2.0的源代码。

```bash
curl https://github.com/tmux/tmux/releases/download/2.0/tmux-2.0.tar.gz -o tmux-2.0.tar.gz
```

tmux依赖于libevent，最新版2.0.22还是2014年初释出的，那就下载这个。

```bash
curl https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz -o libevent-2.0.22-stable.tar.gz
```

## 编译

解压缩后开始编译，首先是libevent：

```bash
cd libevent-2.0.22
./configure --prefix=/home/iceway/temp/
make
make install
```

tmux:

```bash
cd tmux-2.0
./configure --prefix=/home/iceway/temp/ --enable-static CFLAGS=-I/home/iceway/temp/include LDFLAGS=-L/home/iceway/temp/lib64
make
make install
```

## 使用

把编译的tmux独立文件拷贝到服务器上，运行时提示内核版本太久，这下没办法了，连安装软件的权限都没有，还能换内核吗？算了，先用tmux 1.3，等哪天服务器升级了可能会好点。这篇文章就算是练习静态编译了。
