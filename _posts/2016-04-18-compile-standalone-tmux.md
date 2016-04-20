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

> 上面的FDFLAGS指定的目录要根据实际环境确定，有的系统上可能是lib而不是lib64。

## 使用

把编译的tmux独立文件拷贝到服务器上，~~运行时提示内核版本太旧~~（问题已解决，请参见下面的更新）。

-----

## 更新（2016-4-20）

今天看到一片文章[编译器的工作过程](https://github.com/100steps/Blogs/issues/2)，才知道原来静态编译会依赖内核版本，一下子想到这个问题。以前对编译过程没怎么了解，只知道动态编译的可执行文件体积小，运行时依赖动态库；静态编译的可执行文件体积大，但是运行时不依赖动态库。看到这里才想到是因为编译环境和运行环境的差异导致tmux提示内核版本太低。

Linux下用file命令可以看到静态链接的可执行文件依赖的内核版本，如下：

```bash
➜  tmux-2.0 file tmux
tmux: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.0.0, BuildID[sha1]=c33d1f0fd001385a52db0aafdf3827509a8e728f, not stripped
```

看了看工作环境的内核版本是2.6.38的，而我的笔记本上是4.1.15的，不过静态编译的tmux需要的内核是3.0.0的，看来静态编译过程中依赖的内核也不是编译环境所在的内核，应该是由编译器和基本的链接库决定的。

把libevent和tmux的源代码传到工作服务器上，重新编译后再用file查看：

```bash
$ file tmux
tmux: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.15, not stripped
```

现在编译的tmux独立文件可以正常使用，关键是我的配置文件不用再做任何更改了。
