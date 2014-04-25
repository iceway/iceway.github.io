---
layout: post
title:  "Openwrt（1）--代码目录结构"
date:	2014-04-17 14:10:28 +0800
categories: Openwrt
tags: Openwrt
---

从官网上下载的Openwrt代码，一般默认的目录结构如下：

	.
	├── BSDmakefile
	├── Config.in
	├── docs/
	├── feeds.conf.default
	├── include/
	├── LICENSE
	├── Makefile
	├── package/
	├── README
	├── rules.mk
	├── scripts/
	├── target/
	├── toolchain/
	└── tools/

	7 directories, 7 files

## 关键的文件和目录

* **"Makefile"**、**"rules.mk"**、**"include/"**包含了整个Openwrt的基本Makefiles，其中定义了很多的Makefile宏以及相关变量。
* **"package/"**目录包含了Openwrt提供的基本软件包，如需添加自己的软件包，可以参考该目录下的子目录添加相关文件。注意，这些包并不一定在Openwrt中维护，但是可以通过Openwrt提供的脚本快速获取。
* **"scripts/"**目录包含了Openwrt提供的一些实用功能，如获取软件包并检查其完整性、更新所有模块代码到最新版本等。
* **"target/"**目录中的linux下包含了针对不同平台的内核补丁以及特殊配置等。
* **"toolchain/"**和**"tools/"**目录包含一些通用的命令和编译工具链，用来生成固件，编译器和C库。

## 编译中生成的关键目录

* 编译过程中下载的编译链、目标平台和包都会放在一个自动创建的目录**"dl/"**下。注意这个目录在`make distclean`时会自动删除，如果你的网络不是很好或者为方便期间，可以将第一次编译过程中下载的整个dl目录备份，以后`make distclean`删除该目录后只需再次创建一个软链接即可，免去了浪费在下载上的大量时间。
* **"toochian/"**和**"tools/"**目录编译生成结果会存储在以下三个目录中：**"build_dir/host/"**是一个临时目录，用于存储不依赖于目标平台的工具；**"build_dir/toolchain-<arch>\*/"**目录用于存储依赖于指定平台的编译链；**"staging_dir/toolchain-<arch>\*/"**目录是编译链的最终安装位置。
* **"bin/"**目录存储了编译生成的最终固件以及各个包的ipk等。
* **"build_dir/linux-\*/"**目录存储编译过程中展开的内核和驱动模块的代码，以及编译生成的目标文件等。
* **"build_dir/target-<arch>\*/"**目录存储编译过程中展开的所有包的代码，以及编译生成的目标文件等。
