---
layout: post
title:  OpenWrt学习笔记（一）了解OpenWrt
date:	17 Mar 2017 14:57:41 +0800
categories: Openwrt
tags: Openwrt
---

本文是OpenWrt学习笔记第一篇，通过[OpenWrt's build system - About](https://wiki.openwrt.org/about/toolchain)内容来了解OpenWrt系统。

首先来看OpenWrt官网上的基本介绍：

> The OpenWrt build system is a set of Makefiles and patches that allows users to easily generate both a cross-compilation toolchain and a root filesystem for embedded systems. The cross-compilation toolchain uses musl, a tiny C standard library.

由此可见，OpenWrt是一套由许多Makefiles和Patches组成的构建系统，让用户可以轻易的生成一套跨平台编译（交叉编译）的工具链以及一个用于嵌入式设备的根文件系统。

这里说到交叉编译工具链用的是musl C标准库，可能是现在的新版本已经切换到musl，我以前用的版本使用的是uClibc库。

## 编译工具链

编译工具链是一套用于针对你的系统编译源代码的工具集合，包括：

* 一个编译器（OpenWrt用的是gcc）
* 二进制文件编辑工具，如汇编器和链接器（OpenWrt用的是binutils）
* 一个C语言标准库（如GNU libc，musl-libc，uClibc或dietlibc）

## 交叉编译

一般的开发所使用的主机平台和所开发软件所运行的平台是相同的，常见的是在X86平台上开发X86平台软件。但是对于嵌入式系统，大多情况下开发平台和最终软件所要运行的平台是不同个，比如要在X86平台上开发出运行在ARM平台的软件。交叉编译指的就是这种情况，在一个平台上编译用于运行在另一个平台上的代码，谓之交叉编译。

OpenWrt构建系统自动处理编译任务，可以工作在大多数嵌入式系统的指令集架构上，同时也可以让用户手动配置并编译自己的软件。

*OpenWrt的Makefile有自己的语法，不同于Linux make工具的常规Makefile。* OpenWrt的Makefile定义了package的元信息：在哪里下载这个package、如何编译、编译好的二进制文件安装到哪里，等等。

## OpenWrt构建系统 - 特性

* 容易导入软件
* 使用kconfig（Linue Kernel menuconfig）配置不同特性
* 提供整合的交叉编译链（gcc, ld, ...)
* 为autotools、cmake和scons提供抽象化的概念
* 处理标准的下载、打补丁、配置、编译和打包等工作流程
* 对不太规范的package提供若干修正

简单的理解为：OpenWrt包含了交叉编译链工具、大量的makefile文件、以及一些脚本工具等，可以自动完成从准备源代码到编译打包等过程，构建出内核和各种功能软件包，并编译出最终的嵌入式系统可用固件。且用户可以通过kconfig自由配置不同的功能，并能轻易将自己的软件导入到OpenWrt构建系统中。

##  OpenWrt构建系统 - Make目标

* 提供多个高层次目标给标准的package工作流程
* 目标统一格式为`component/name/action`，如`toolchain/gdb/compile`或`package/mtd/install`
* 准备package的源码树目标，如：`package/foo/prepare`
* 编译一个pacakge的目标，如：`package/foo/compile`
* 清除一个package的目标，如：`package/foo/clean`

OpenWrt提供了统一的目标格式，对于所有的package、内核等都可以通过通用目标格式`component/name/action`来执行特定的make目标，其中的action部分包括：

| Action | Usage |
|---------|---------|
| `prepare` | 准备源码树 |
| `compile` |  编译源码 |
| `install` | 安装编译好的最终文件 |
| `clean` | 清除编译的目标文件 |

这些目标现在可能不容易理解，后面对OpenWrt了解的多了再回头看就很容易理解。因为OpenWrt的高度集成化，在通过OpenWrt构建一个固件时，提前配置后只要执行命令`make V=s`，后面的过程将由OpenWrt自动完成。有时修改某个package后想尽快看看改动效果，如果还是通过`make V=s`会花费更多时间构建整个项目，这时就可以按照上面的目标来单独处理。比如只想单独编译内核，就可以执行`make target/linux/compile V=s`；再如想独立准备dnsmasq的源码树，执行`make package/dnsmasq/prepare V=s`即可。

## OpenWrt构建系统 - 构建顺序

1. tools： automake、autoconf、sed、cmake
2. toolchain/binutils：as、ld，...
3. toolchain/gcc：gcc、g++、cpp，...
4. target/linux：内核模块
5. package： 核心pacakge
6. target/linux：内核镜像
7. target/linux/image：最终的固件文件

这里的构建顺序对理解OpenWrt整个系统的结构很有帮助，再后面深入学习OpenWrt的结构后应该再回头看看。

## 补丁管理

大多数的原生package都不能直接像其期望那样工作，需要针对具体平台甚至时编译过程打补丁。OpenWrt集成了[quilt](https://en.wikipedia.org/wiki/Quilt_(software))来管理补丁。

针对补丁管理，OpenWrt提供了几个make目标来实现，包括`prepare`、`update`和`refresh`，具体的用法主要依赖于quilt工具，这里不在多说，需要深入了解的话，可以参考[OpenWrt补丁管理](https://wiki.openwrt.org/doc/devel/patches)。

值得一提的是，OpenWrt对于添加的内核补丁，有一些命名上的约定应该遵守：

> 在target/linux/generic或具体平台的中，`patches_*`子目录包含了内核补丁，所有内核补丁应该都命名为'NNN-lowercase_shortname.patch'形式，并按照如下分类排序：
> 
> ```
> 0xx - 上游更新
> 1xx - 等待上游合并的代码
> 2xx - 内核的构建、配置及头文件等补丁
> 3xx - 平台特定的补丁
> 4xx - mtd相关补丁，子系统及驱动
> 5xx - 文件系统相关补丁
> 6xx - 通用网络补丁
> 7xx - 网络、物理网卡驱动等补丁
> 8xx - 其他驱动补丁
> 9xx - 无分类的其他补丁
> ```
