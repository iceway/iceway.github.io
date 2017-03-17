# OpenWrt学习笔记（二）安装及使用

本文学习[OpenWrt安装](https://wiki.openwrt.org/doc/howto/buildroot.exigence)和[OpenWrt使用](https://wiki.openwrt.org/doc/howto/build)两篇文章。

按照官方描述，OpenWrt构建系统是用于构建OpenWrt Linux发行版的基础系统，工作在GNU/Linux、BSD及MAC OSX操作系统上。官网上也说了OpenWrt要求系统必须是大小写区分的，而Windows系统对文件名等都不区分大小写，所以估计是不能在Windows下开发的。

## 在GNU/Linux上安装过程

在开始安装前，官网上给出了几点警示信息：

> 1. 所有的命令都用非root用户执行；
> 2. 任何构建命令都要在OpenWrt构建系统代码的根目录下执行；
> 3. 不要在完整路径中包含空格的目录下构建；
> 4. 将整个OpenWrt目录的属主改成你自己，而不是root用户。

现在，OpenWrt的代码已经转移到github上保管，所以将要安装git来获取OpenWrt所有原始文件。还有一些编译工具也需要安装到系统中，以及某些模块可能是通过subversion或mercurial获取代码的，这些工具都要安装。在你的Linux系统上安装：

```
git git-core build-essential libssl-dev libncurses5-dev unzip gawk zlib1g-dev subversion mercurial
```

下载OpenWrt原始文件仓库，通过以下命令：

```
git clone https://github.com/openwrt/openwrt.git
```

这个仓库中，分支master就是OpenWrt的trunk分支，而其他的分支对应了当前OpenWrt还在活跃开发中的版本，分支没有的早期版本需要查找官方文档去下载。

接下来这步是可选的，不一定要执行。下载和安装所有可用的feeds（关于feeds，后面在具体讲，这里可以先理解为feeds是一些额外扩展的pacakge）：

```
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
```

让OpenWrt构建系统检查丢失的package，使用下面的*任意一个*命令，并选择你要启用的功能。

```
make menuconfig #(most likely you would like to use this)
make defconfig
make prereq
```

接下来就可以通过`make`命令去构建OpenWrt固件了。

> OpenWrt需要的软件包在不同的Linux系统上，名字可能会有差异，[这个表格](https://wiki.openwrt.org/doc/howto/buildroot.exigence#table_of_known_prerequisites_and_their_corresponding_packages)列出了常见的Linux系统上对应的软件包名字，对照此表安装所需软件即可。
