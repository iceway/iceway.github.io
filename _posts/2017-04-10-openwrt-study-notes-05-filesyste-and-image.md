---
layout: post
title:  OpenWrt学习笔记（五）文件系统和固件
date:	10 Apr 2017 15:26:18 +0800
categories: Openwrt
tags: Openwrt
---

本文学习内容参见[filesystems](https://wiki.openwrt.org/doc/techref/filesystems)，[image](https://wiki.openwrt.org/doc/techref/image.makefile)

本文介绍关于在内置的flash上的OpenWrt安装的文件系统。关于将文件系统安装到其他设备，包括分区和挂载的方式，请参见[General Storage](https://wiki.openwrt.org/doc/howto/storage)。


## 常见的文件系统

* OverlayFS

  这个文件系统将两个文件系统合并在一起，一个只读的和一个可写的文件系统。这样做的好处在于，可以将固件文件系统的初始内容作为其中的只读文件系统，而将系统启动后的改动内容保存在可写文件系统中。由于两个文件系统是合并在一起的，设置可写文件系统的文件优先，在需要恢复出厂值的时候，只需要删除掉可写文件系统中的内容即可。

* tmpfs

  名字已经反映出这个文件系统的作用，是一个临时文件系统。这种文件系统没有设计负载均衡，并且是一种易失性的文件系统（也就是说设备重启后，这中文件系统中的内容将会丢失）。所以一般，/tmp将会挂载为tmpfs格式，且/var会软连接到其上。/dev位于自身的一个小型tmpfs分区。

* SquashFS

  这是一个只读文件系统，其优点在于极高的压缩比。可用gzip压缩内容，不过OpenWrt中用了更高压缩比的lzma格式。由于是只读系统，不需要将数据对齐，允许将数据已更紧密的格式打包，这意味着将比JFFS2更加节省空间（平均能节省20%到30%的空间）。

  > 注意：在NAND闪存上用SquashFS时存在一个共知的问题：SquashFS没有坏块管理并要求所有块都是有序的，但是本身NAND的坏快管理要能跳过坏块，且偶尔还要重映射这些块。这就是为何在NAND上用原始的SquashFS是一个坏主意的原因。

* JFFS2

  这个文件系统是一个使用lama压缩格式的可写文件系统，支持日志记录和负载均衡。

* UBIFS

  UBIFS是一种针对原始flash的文件系统，在最新的OpenWrt NAND目标中使用。

## 在OpenWrt上的实现

OpenWrt使用了SquashFS和JFFS2文件系统，将这两者结合组成一个OverlayFS的文件系统。在原始闪存中，内核与这三种分区是独立存放的，此外编译时，内核就已经通过LZMA和gzip压缩过。

下图是从[Partitioning of Flash](https://wiki.openwrt.org/doc/techref/flash.layout#partitioning_of_the_flash)复制的，可以看到OpenWrt中默认的flash布局及其对应的文件系统。

<table class="inline">
	<tbody><tr class="row0">
		<th class="col0"> Layer0 </th><td class="col1 centeralign" colspan="6">  raw flash  </td>
	</tr>
	<tr class="row1">
		<th class="col0"> Layer1 </th><td class="col1 centeralign" rowspan="3">  bootloader <br>
partition(s)  </td><td class="col2 centeralign" rowspan="3">  optional <br>
SoC <br>
specific <br>
partition(s)  </td><td class="col3 centeralign" colspan="3">  OpenWrt firmware partition  </td><td class="col6 centeralign" rowspan="3">  optional <br>
SoC <br>
specific <br>
partition(s)  </td>
	</tr>
	<tr class="row2">
		<th class="col0"> Layer2 </th><td class="col1 centeralign" rowspan="2">  Linux Kernel  </td><td class="col2 centeralign" colspan="2">  <strong><code>rootfs</code></strong> <br>
mounted: "<code>/</code>", <a href="https://wiki.openwrt.org/doc/techref/filesystems#overlayfs" class="wikilink1" title="doc:techref:filesystems">OverlayFS</a> with <code>/overlay</code>  </td>
	</tr>
	<tr class="row3">
		<th class="col0"> Layer3 </th><td class="col1 centeralign">  <strong><code>/dev/root</code></strong> <br>
mounted: "<code>/rom</code>", <a href="https://wiki.openwrt.org/doc/techref/filesystems#squashfs" class="wikilink1" title="doc:techref:filesystems">SquashFS</a> <br>
size depends on selected packages  </td><td class="col2 centeralign">  <strong><code>rootfs_data</code></strong> <br>
mounted: "<code>/overlay</code>", <a href="https://wiki.openwrt.org/doc/techref/filesystems#squashfs" class="wikilink1" title="doc:techref:filesystems">JFFS2</a> <br>
"free" space  </td>
	</tr>
</tbody></table>

### 文件系统的启动

OpenWrt详细的启动过程会在稍后的笔记中介绍，这里只关注于文件系统的启动过程。

1. 内核从一个已知的原始闪存分区（没有文件系统，可理解为裸设备）启动，之后运行的内核会扫描rootmfs这个mtd分区查找一个有效的超级块，并挂载这个SquashFS分区（这个分区包含了/etc），挂载后执行`/etc/preinit`。
2. `/etc/preinit`会执行`/sbin/mount_root`。
3. `/sbin/mount_root`挂载JFFS2分区到/overlay并联合SquashFS分区（默认是挂载到/rom）创建出一个新的虚拟根文件系统（作为`/`挂载）。
4. 接下来的启动过程将会由`/sbin/init`继续。

> 以前的OpenWrt中overlayfs的挂载点是/jffs2而不是/overlay。

### 详细解释

SquashFS和JFFS2都是使用LZMA压缩过的文件系统，SquashFS是一个只读的文件系统，而JFFS2是一个可写的文件系统且包含日志记录以及负载均衡。

在写入固件时我们的任务是尽量在SqushFS中放入尽可能多的常用功能同时又不浪费空间在非必须特性上。附加的功能可以由用户安装到JFFS2上。使用`mini_fo/overlayfs`意味着对于用户而言，文件系统作为一个大的可写文件系统出现，在SquashFS和JFFS2之间感受不到分割界限。当写入数据时这些文件就被简单的复制到JFFS2文件系统中。

我们在flash中如此紧密的塞满内容的真相是，如果固件发生改变，JFFS2分区的大小和位置也会跟着改变，可能会擦除一大块JFFS2数据并导致文件系统损坏。为了处理这个问题，我们已经实现了一个策略：在每次重写后JFFS2数据就被重新格式化。实现这些的技巧在于一个特殊值，0xdeadc0de；当一个JFFS2分区中出现这个值时，所有重这个位置到分区结束位置的数据被认为是已擦除的。因此，固件镜像尾部是值0xdeadc0de，从这个位置开始就变成了JFFS2分区的开始。

实际上，我们使用一个联合的、压缩过的且只有部分只读的文件系统也对包管理有一些有趣的影响：特别的，你要对你更新的包要特别小心。OPKG命令更喜欢将更新包安装到JFFS2上，但是却不能从SquashFS上移除原始的包；结果就是你慢慢的使用了越来越多的空间直到JFFS2分区填满。opkg工具无法知道在JFFS2分区上还有多少可用空间，由于这个分区是压缩过的，因此它将会盲目的前进直到opkg系统挂掉，这时你将只有非常少量的空间可用，甚至你可能都无法再通过opkg删除任何东西（因为opkg命令执行时本身也需要占用空间）。

很多嵌入式目标中都使用NOR flash写入根文件系统，OpenWrt实现了一个很巧妙的方式充分使用有限的闪存能能力，同时对用户而言又保持了灵活性：基本上，在创建固件期间，所有文件系统的内容都被打包进一个SquashFS文件系统。关于这中想法有一个重要的细节：这是一个只读文件系统。为克服这个限制，OpenWrt使用NOR根系统剩下的空间存储了一个附加的读写文件系统（JFFS2），它覆盖在根文件系统上（意味着，允许从SquashFS中读取所有未更改的文件，而将所有修改过的文件保存到jffs2部分）。这种设计对用户有一个更重要的优势：即使读写文件已经几乎损坏，用户依然可以启动到failsafe模式（这种模式下只挂载了SquashFS部分）并从这里开始处理。

### 技术细节

是否可以将整个文件系统都切换到JFFS2？

整个根文件系统只包含JFFS2分区是有可能的，优点是改变文件不再需要早只读文件系统中保留一份旧的拷贝，这样可以更节省空间。但是缺点是你将不能启动到failsfae模式，如果JFFS2系统被破坏，系统将很难恢复。并且JFFS2相比SquashFS需要更多的空间。

## 生成固件的Makefile

OpenWrt项目构建会生成两个主要文件，内核镜像和根文件系统镜像，最终会将这两个镜像文件合并成一个固件（firmware）。在你的平台目录下你需要创建一个文件 告诉构建系统如何处理编译好的内核，其实就是告诉构建系统如何生成你的固件文件。其中的大多数工作都由include/image.mk文件自动完成，但是不同平台或一些设备需要一些特定的工作让固件更有用。这就要自动手动添加一些内容到生成镜像的makefile中（如target/linux/ipq806x/image/Makefile）。

### Image/Prepare

这个make宏用于向固件镜像添加数据，但是一般用于将镜像文件移动到其他目录（如$(KDIR)）。

比如：

```makefile
cat $(LINUX_DIR)/arch/arm/boot/zImage >> $(KDIR)/$(call zimage_name,$(1))
```

### Image/Build/Initramfs

这个宏允许在efl文件加载到设备前自动修改elf文件，比如：

```makefile
$(BIN_DIR)/$(IMG_PREFIX)-vmlinux.elf
```

### Image/Build/jffs2-64k, Image/Build/jffs2-64k-128k, Image/Build/squashfs, Image/Build

用于调用其他定义的宏（squashfs，jffs2-64k等），这些调用最终的生成的文件被放在$(TARGET_DIR)目录中。

对这些宏的调用如下：

```makefile
$(call Image/Build/$(1),$(1))
```

### 示例 (文件target/linux/ipq806x/image/Makefile的部分内容)

```makefile
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

UBIFS_OPTS = -m 2048 -e 124KiB -c 4096 -U -F
UBINIZE_OPTS = -m 2048 -p 128KiB

KERNEL_LOADADDR := 0x42208000

define Image/Prepare
        $(CP) $(LINUX_DIR)/vmlinux $(KDIR)/$(IMG_PREFIX)-vmlinux.elf
        mkimage -A arm -O linux -T filesystem -C none \
                -a $(KERNEL_LOADADDR) -e $(KERNEL_LOADADDR) \
                -n 'ARM OpenWrt fakeroot' \
                -s $(KDIR_TMP)/root.dummy-uImage.tmp
        echo -ne '\xff' > $(KDIR_TMP)/root.dummy
        cat $(KDIR_TMP)/root.dummy $(KDIR_TMP)/root.dummy-uImage.tmp > $(KDIR)/root.dummy
endef

define Image/BuildKernel
        $(CP) $(KDIR)/$(IMG_PREFIX)-vmlinux.elf $(BIN_DIR)
endef

define Image/Build/squashfs
        cp $(KDIR)/root.squashfs $(KDIR)/root.squashfs-raw
        $(call prepare_generic_squashfs,$(KDIR)/root.squashfs)
endef

define Image/Build
        $(call Image/Build/$(1),$(1))
        dd if=$(KDIR)/root.$(1) of=$(BIN_DIR)/$(IMG_PREFIX)-$(1)-root.img bs=2k conv=sync
endef
```
