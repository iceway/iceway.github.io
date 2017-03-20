---
layout: post
title:  OOpenWrt学习笔记（二）安装及使用
date:	20 Mar 2017 16:38:16 +0800
categories: Openwrt
tags: Openwrt
---

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

``` bash
git clone https://github.com/openwrt/openwrt.git
```

这个仓库中，分支master就是OpenWrt的trunk分支，而其他的分支对应了当前OpenWrt还在活跃开发中的版本，分支没有的早期版本需要查找官方文档去下载。

接下来这步是可选的，不一定要执行。下载和安装所有可用的feeds（关于feeds，后面在具体讲，这里可以先理解为feeds是一些额外扩展的pacakge）：

```bash
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
```

让OpenWrt构建系统检查丢失的package，使用下面的*任意一个*命令，并选择你要启用的功能。

```bash
make menuconfig #(most likely you would like to use this)
make defconfig
make prereq
```

接下来就可以通过`make`命令去构建OpenWrt固件了。

> OpenWrt需要的软件包在不同的Linux系统上，名字可能会有差异，[这个表格](https://wiki.openwrt.org/doc/howto/buildroot.exigence#table_of_known_prerequisites_and_their_corresponding_packages)列出了常见的Linux系统上对应的软件包名字，对照此表安装所需软件即可。

## OpenWrt构建系统的用法

使用OpenWrt构建系统，请注意这些需求：

1. 在你的系统上安装OpenWrt构建系统及其依赖的软件包。
2. 至少需要有3~4GB的可用磁盘空间。
3. 相关的环境变量：
  * 不能设置`SED`变量。如果设置过此变量，在编译前执行`unset SED`；
  * 变量`GREP_OPTIONS`不能包含`-initial-tab`或其他影响其输出的选项；
  * 将`<buildroot dir>/staging_dir/host/bin`和`<buildroot dir>/staging_dir/toolchain-<platform>-<gcc_ver>-<libc_ver>/bin`添加到你的`PATH`变量最前面，这两个目录在编译前期（toolchain的编译）会自动生成。

### 使用步骤

1. 更新OpenWrt仓库（参考命令`git pull`）。
2. 更新及安装所有的feeds。
3. 配置项目，选中你希望得到的固件中包含的相关package。
4. 开始编译，OpenWrt会自动编译toolchain，交叉编译的源码，选中的package，以及最终生成一个准备好的完整镜像。
5. 安装固件即可（一般可以通过GUI界面升级）。

#### 更新Feeds

要更新feeds，可以在OpenWrt根目录下执行：

```bash
./scripts/feeds update -a
```

这里的更新，只是将feeds中相关包的信息下载下来，但是在OpenWrt的menuconfig中还不能配置，如果要在menuconfig中看到，需要再次执行下面命令之一。在这些命令中，install的含义只是让package在menuconfig可见，并不是实际的安装编译，可以理解为将package信息安装到OpenWrt构建系统中以便后续配置编译等。

```bash
./scripts/feeds install <PACKAGENAME>
./scripts/feeds install -a
```

### 固件镜像的配置

OpenWrt构建系统只要一个简单的make命令就可以自动编译完toolchain、package、kernel并生成最终的固件镜像，而且用户可以自由配置，一般的配置动作如下：

1. 执行`make menuconfig`，并设置“Target System”，“Subtarget”，“Target Profile”；
2. 执行`make defconfig`，这一步会自动根据上一步的配置自动生成对应的默认配置；
3. 执行`make menuconfig`，手动选择并配置可用的package；
4. 执行`scripts/diffconfig.sh >mydiffconfig`，可以将你对配置的改动保存到文件中，避免每次都要重新配置；
5. 最后就是开始构建，执行命令`make V=s`，其中变量V的值s表示构建过程中输出详细的过程信息，`V=99`和`V=s`含义完全相同。

#### menuconfig

OpenWrt构建系统的配置接口可以处理诸如：选择目标平台、选择需要编译的package、选择需要包含到最终固件镜像的pacakge、以及一些内核选项等。执行命令`make menuconfig`后，就打开了配置接口，系统会自动加载你之前配置好的配置文件内容并更新其中对应的package的选中状态，然后就能自由配置OpenWrt系统了。

对于可以配置的package，一般有三种选择：`y`、`m`、`n`三种选项，各自的含义如下：
* 按下<kbd>y</kbd>键（Yes），表示这个package将要被编译，并包含到最终的固件镜像中。
* 按下<kbd>m</kbd>键（Module），表示这个pacakge将要被编译，但是*不会*包含到最终的固件镜像中。只是将编译后的文件打包成ipk格式的安装包，稍后可以在烧入固件并启动后根据需要通过opkg工具安装这个ipk包。
* 按下<kbd>n</kbd>键（No），表示不会编译这个package。

大多数Linux发行版都是直接提供编译好的系统安装镜像，里面包含了很多该发行版的默认包含的已编译好的软件（也有需要自己从源代码开始编译的发行版）。安装好后，除了默认已经集成在安装镜像中的软件，还可以通过包管理器（如Debian的apt-get，Redhat的yum，OpenSUSE的zypper等）安装一些已经编译并打包好的软件。上述配置的选项和这个情况类似，OpenWrt最终会自动生成一个固件镜像，可以将其理解为Linux的发行版镜像，选择y的package会被自动编译并包含在生成的镜像中；选择m的package会被自动编译，并打包成ipk格式的安装包，但是不会包含在固件镜像中，稍后启动启动后可以通过OpenWrt的包管理工具（opkg）去安装这些软件。

至此，已经介绍了如何从头开始配置并编译OpenWrt系统，一个标准的配置过程将会包含下列修改：

1. 配置目标系统平台（Target System）；
2. 选择所需的软件包（Package Selection）；
3. 构建系统的设置（Build System Setting），主要针对编译工具及编译过程做设置；
4. 内核模块（Kernel Modules）。

#### kernel_menuconfig

上面说了可以通过`make menuconfig`配置很多功能，包括选择内核模块，但是Linux内核也有复杂而庞大的配置参数可以自由配置，对此OpenWrt单独准备了另一个make目标来针对内核单独配置：

```bash
make kernel_menuconfig CONFIG_TARGET=<subtarget>
```

其中的`<subtarget>`是`target/linux`目录下的可选平台之一，如x86等。如果之前已经通过menuconfig在Target System中配置过并保存到`.config`文件中，那就不用在命令行指定`COFNIG_TARGET`变量了。

内核中可配置项的`y`、`m`、`n`三种选项含义和标准内核配置相同，特别是`m`表示某个功能被编译成内核模块，而不是像menuconfig中表示的只编译而不包含在固件中。

#### 配置时使用diff文件

除了使用menuconfig的方式配置OpenWrt项目，还可以使用配置项的diff文件来配置。这个diff文件仅包含了你的配置项与默认配置项的差异部分。采用这种方式的一个好处是可以在你的OpenWrt项目上使用版本控制工具追踪，并且收到OpenWrt更新的影响更少，因为这个diff文件仅包含了改动部分。

具体用法如下：

1. 创建diff文件
  ```bash
  ./scripts/diffconfig.sh > diffconfig
  ```
2. 使用diff文件
  ```bash
  cp diffconfig .config # write changes to .config
  make defconfig # expand to full config
  ```

### 构建固件镜像

配置好OpenWrt后，只需要执行命令`make`就可以开始自动构建整个项目了，命令`make V=s`可以输出详细的构建过程。

上一篇中已经说过了一些单独编译某个package的技巧，这里简单举例如下，比如要清除、编译并安装cups包，可以执行如下命令：

```bash
make package/cups/{clean,compile,install} V=s
```

### 固件镜像位置

当整个构建过程成功完成后，OpenWrt最终自动生成了固件镜像。最终生成的镜像文件位于目录`<buildroot_dir>/bin/`下，并且是保存在一个以对应的模目标平台命名的子目录下。比如针对ar71xx设备构建的固件将会位于`<buildroot_dir>/bin/ar71xx/`目录。

### 清除编译好的目标

OpenWrt提供以下几种make目标用于清除。

1. `make clean`：删除目录`<buildroot_dir>/bin/`和`<buildroot_dir>/build_dir/`内的所有内容，不会删除toolchain以及你在配置文件`.config`中选中的系统架构平台目标，当然也不会删除在构建过程中根据你的配置自动下载的pacakge的源码包，以及下载的feeds。
2. `make dirclean`：删除目录`<buildroot_dir>/bin/`和`<buildroot_dir>/build_dir/`内的所有内容，以及`<buildroot_dir>/staging_dir/`、`<buildroot_dir>/toolchain/`和`<buildroot_dir>/logs/`的内容。
3. `make distclean`：删除所有已经编译和配置的文件，同时会删除所有下载的feeds和pacakge源码。另外，该命令也会删除你的配置文件`.config`，所以如果没有保存你的配置之前，一定要慎用此命令。

当然了，也可以只清除某个package或者部件的编译内容，如：

```bash
make target/linux/clean
make package/base-files/clean
make package/luci/clean
```
