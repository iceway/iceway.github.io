---
layout: post
title:  OpenWrt学习笔记（六）启动过程
date:	12 Apr 2017 15:24:38 +0800
categories: Openwrt
tags: Openwrt
---

本文学习内容[The Boot Process](https://wiki.openwrt.org/doc/techref/process.boot)。

## 启动“三人组”

设备上电后非常基础的底层硬件就已准备好了，你可以通过JTAG口连接到设备上并且可以执行一些命令（比如通过JTAG写入bootloader image）。一般而言到这里就直接开始执行bootloader了（对于NOR flash，由于和总线统一编址，可以直接让CPU从NOR flash的初始位置开始执行；而对于NAND flash，CPU上电准备时就会自动将NAND Flash最前面的一小段内容读取到CPU的一个内置缓存上执行，然后这一小段代码再继续读取完整的bootloader到内存中并执行）接下来就开始执行以下三大部分。

### Bootloader

1. 闪存中的bootloader开始执行。
2. bootloader执行POST （Power On Self Test），上电后自检，这一布初始化一些底层硬件。
3. bootloader从闪存中读取内核镜像（bootloader已知内核镜像在闪存中的位置），解压缩内核镜像到运行内存中。
4. bootloader开始执行内核，并传入参数`init=...`（默认情况下init的值是/etc/preinit)。

至此，bootloader的启动过程就结束了，控制权开始转给内核。

### Kernel

1. 内核进一步启动自身（主要是执行架构相关的汇编代码）。
2. 执行命令`start_kernel`，开始正式进入内核初始化阶段。
3. 内核扫描mtd分区`rootfs`寻找一个有效的额超级块，一旦找到了就挂载这个SquashFS文件系统分区（这个分区包含了/etc目录）。
4. 执行/etc/preinit，做一些预初始化的设置（比如创建目录，挂载proc和sys伪文件系统等）。
5. 内核挂载其他分区（如JFFS2）到根文件系统下。
6. 如果INITRAMFS没有定义，就调用/sbin/init命令（这个最终就是用户空间的第一个进程，可以看作是所有其他进程的父进程）。
7. 最后一些内核线程变成了用户空间的init进程。

到这里，内核的启动过程就完了，接下来就要执行一些用户空间定义的自启动脚本。

### init

用于空间开始于内核挂在了根文件系统且最早的程序被运行（默认是/sbin/init）。记住，应用程序和内核之间的接口是C库及其提供的系统调用。

1. init读取/etc/inittab文件，找到sysinit入口（默认是`::sysinit:/etc/init.d/rcS S boot`).
2. init调用`/etc/init.d/rcS S boot`。
3. rcS执行真实文件位于`/etc/rc.d/S##xxxxxx`的启动脚本的软连接，并传入参数start。
4. 当整个rcS执行完成后，系统已经启动并正常运行了。

启动过程的三大部分到这里就完成，系统也正常启动并运行，下面再看看其他一些细节。

### 普遍的启动脚本

> 注意：通过opkg安装的一些包可能会添加额外的启动脚本。

| 文件 | 说明 |
|-----|-----|
| S05defconfig | 如果配置文件不存在，针对平台创建默认值配置文件，在OpenWrt安装后第一次启动会实际执行该动作（实际上就是将/etc/config目录下不存在的文件从/etcdefconfig/$board目录下拷贝过来） |
| S10boot | 启动hotplug-script，挂载文件系统，启动syslogd等 |
| S39usb | 挂载usbfs: mount -t usbfs none /proc/bus/usb |
| S40network | 启动网络子系统（运行/sbin/netifd，开启以太网接口，无线网络 |
| S45firewall | 从配置文件/etc/config/firewall中创建防火墙规则 |
| S50cron | 启动crond进程 |
| S50dropbear | 启动dropbear进程 |
| S50telnet | 检查root的密码，如果没有设置，则启动/usr/sbin/telnetd进程 |
| S60dnsmasq | 启动dnsmasq进程　|
| S95done | 执行/etc/rc.local脚本 |
| S96led | 加载led配置（/etc/config/system）并设置led灯 |
| S97watchdog | 启动看门狗守护进程（/sbin/watchdog）|
| S99sysctl | 解释/etc/sysctl.conf文件并设置 |

init进程在系统启动后将会一直运行。当执行shutdown命令时，init进程将会：

1. 读取/etc/inittab查找shutdown（默认是`::shutdodwn:/etc/init.d/rcS K stop`）。
2. init进程调用`/etc/init.d/rcS K stop`。
3. rcS执行所有位于/etc/rc.d/K##xxxxxx的关闭脚本，并传入参数stop。
4. 系统停止或重启。

| 文件 | 说明 |
|-----|-----|
| K50dropbear | 杀死所有dropbear的实例进程 |
| K90network | 停用所有的网络接口，并停止netifd |
| K98boot | 启动logger守护进程 |
| K99umount | 将缓存写入磁盘（闪存），卸载所有的文件系统 |

## 启动顺序的细节

### bootloader

以grub为例，bootloader初始化后，选择任一启动菜单上出现的启动项，bootloader将会加载内核，比如openwrt-x86-ext2-image.kernel启动菜单正常启动的加载参数如下：

```
kernel /boot/vmlinuz root=/dev/hda2 init=/etc/preinit [rest of options]
```

这个启动项告诉grub内核位于/boot目录下，文件名是vmlinuz。这行的其他选项将会传递给内核。内核启动时接收的启动参数，都可以在文件/proc/cmdline文件中看到。

### /etc/preinit

preinit脚本的主要目的是初步就i安插并设置其他的启动脚本。一个主要的任务是挂载/proc和/sys伪文件系统，以便能访问系统信息以及让一些控制功能可用。另一个重要的功能是准备/dev目录（包括创建具体的设备文件），这样就能访问如console等的设备。preinit的最终任务就是启动init进程本身。

### busybox的init

init进程被认为是所有其他进程的父进程，因此它控制诸如启动守护进程、修改运行级别、配置console或tty设备，就像其他的管理任务一样。

一旦init进程启动，它会读取/etc/inittab配置文件，这个配置文件告诉它应该启动哪些进程，针对确定活动的监视器，以及什么时候触发一个活动调用相关程序。

busybox使用的init程序是一个极简的守护进程。它不包含运行级别的信息，因此这个配置文件只有普通init配置文件的部分属性。如果你运行的是一个完整的linux桌面，可以通过命令`man inittab`了解普通init进程及其入口。配置文件中每一行的各个字段以冒号分割，定义如下：

```
[ID] : [Runlevel(s)] : [Action] : [Process to execute ]
```

对于busybox的init，只需要上面的第一（ID），第三（Action）和第四（Process）字段。busybox的init对于普通init的几个特定说明：ID字段用于控制TTY/Console，没有定义运行级别。一个极简的/etc/inittab看起来如下：

```bash
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K stop
tts/0::askfirst:/bin/ash –login
ttyS0::askfirst:/bin/ash –login
tty1::askfirst:/bin/ash –login
```

前两行ID字段为空表示他们对任何终端或console没有特殊含义，其他行被定向到特定的终端或console。

注意sysinit和shutdown的动作调用的是同一个程序（/etc/init.d/rcS脚本文件）。唯一的区别是传递给rcS的参数。

## /etc/init.d/rcS脚本

到这个程序执行时，基本的设置都已经完成了，所有系统文件或者配置文件都可以访问了，现在可以开始启动其他进程了。

rcS脚本是优美而简单的：其唯一目标就是执行所有位于/etc/rc.d目录下的脚本，并传入适当参数。如果你关注的是sysinit入口，rcS脚本会被传入参数“S”和“boot”并调用。因此调用rcS执行时，会将下面代码中的$1替换成“S”，将$2替换成“boot”并执行：

```bash
for i in /etc/rc.d/$1* ; do
        [ -x $i ] && $i $2
done
```

仔细看/ect/rc.d目录，可能会注意到有些脚本有相关的启动脚本（S开头的文件名）链接，但是没有关闭相关的脚本（K开头的文件名）链接（如/etc/init.d/httpd）。而另一些却没有启动脚本的链接，而只有关闭脚本的链接（如/etc/init.d/umount）。

对于httpd而言，它的进程是否关闭并没有影响，在系统关闭前没有什么需要清理的东西。

但是，在系统关闭前必须执行umount脚本以保证所有数据都从缓存中刷新到实际的存储媒体中。否则就可能出现数据损坏或其他异常。而这个动作在系统启动时并不需要执行，因为这些存储媒体在其他地方就会被挂载（如/etc/preinit），因此就不需要对其创建启动相关的脚本。
