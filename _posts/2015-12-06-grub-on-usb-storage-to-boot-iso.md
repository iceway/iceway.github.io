---
layout: post
title:  "U盘安装GRUB2引导多个Linux ISO镜像"
date: 09 Dec 2015 09:06:55 +0800
categories: 系统
tags: GRUB
---

## USB多启动工具盘

GRUB是一个很强大的多系统启动引导工具，甚至可以直接引导启动磁盘上的ISO镜像文件。如果把GRUB安装到U盘中，然后再把系统ISO镜像也拷贝到U盘上，就可以做一个可移动的U盘安装盘，要是把类似puppy这样的系统ISO文件也放在U盘上，就是一个移动的操作系统了。

我手上有一个东芝的8G U盘，打算把openSUSE 42.1的ISO镜像和Puppy 7.0.3的ISO镜像都放在这个U盘上，同时在U盘上安装GRUB2，可以通过GRUB2分别启动这两个系统盘，同时U盘还可以当作正常的移动存储设备在Windows/Linux下使用。

> 以下所有操作是在openSUSE 13.2系统上完成，操作前先备份U盘上的重要数据。

## 分区

U盘插入电脑后，先找到U盘的设备名。

1.  运行命令`ls /proc/scsi/usb-storage/`，这个目录下有一个文件名为数字的文件，记下这个数字，假设为N。
2.  运行命令`ls /sys/class/scsi_device/`，这里应该可以看到一个名为`N:0:0:0`的文件夹（其中N是第一步中看到的数字）。
3.  运行命令`ls /sys/class/scsi_device/N:0:0:0/device/block/`，这里显示的文件夹名字就是该U盘对应的设备名。
4.  那么该U盘的完整设备路径就是`/dev/XXX`，其中`XXX`是第三步看到的设备名。

在我的电脑上操作看到的设备如下：

```
[root@iceway-pc iceway]$ ls /proc/scsi/usb-storage/
10
[root@iceway-pc iceway]$ ls /sys/class/scsi_device/10\:0\:0\:0/device/block/
sdc
[root@iceway-pc iceway]$ ls -l /dev/sdc
brw-rw---- 1 root disk 8, 32 Dec  6 12:22 /dev/sdc
```

运行parted工具给U盘分区，执行命令`parted /dev/sdc`，进入parted交互接口，把U盘分成两个主分区，一个分区安装GRUB并作为启动分区，使用EXT3文件系统；另一个作为正常的数据盘，为了能在Windows下正常使用，这个分区用NTFS文件系统。

```
[root@iceway-pc iceway]$ parted /dev/sdc
GNU Parted 3.1
Using /dev/sdc
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mktable msdos                                                    
Warning: The existing disk label on /dev/sdc will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? Yes                                                               
(parted) mkpart primary ntfs 0% 96%
(parted) mkpart primary ext3 96% 100%
(parted) set 2 boot on                                                    
(parted) print                                                            
Model: TOSHIBA TransMemory (scsi)
Disk /dev/sdc: 7756MB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  7446MB  7445MB  primary               type=07
 2      7446MB  7755MB  309MB   primary               boot, type=83

(parted) quit
Information: You may need to update /etc/fstab.

[root@iceway-pc iceway]$ 
```

格式化分区。

```
[root@iceway-pc iceway]$ mkfs.ntfs -L DATA /dev/sdc1 
Cluster size has been automatically set to 4096 bytes.
Initializing device with zeroes: 100% - Done.
Creating NTFS volume structures.
mkntfs completed successfully. Have a nice day.
[root@iceway-pc iceway]$ mkfs.ext3 -L boot /dev/sdc2 
mke2fs 1.42.12 (29-Aug-2014)
Creating filesystem with 302080 1k blocks and 75776 inodes
Filesystem UUID: 9b6e6106-03b9-48ca-970d-39f67c2d1c1b
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

[root@iceway-pc iceway]$
```

## 安装GRUB2

GRUB2软件自带一个工具`grub2-install`可以将GRUB2安装到指定目录，`grub2-install --help`可以看到该工具的详细用法。

要把GRUB2安装到U盘的第二个分区中，要先将该分区挂载到某个目录，然后执行grub2-install并指定`boot-directory`为具体的挂载目录安装；命令最后指定要将GRUB2的主引导分区安装到哪个设备，这里是`/dev/sdc`表示U盘设备（注意不是U盘的某个分区，而是整个U盘设备）。

```
[root@iceway-pc iceway]$ mkdir /tmp/DATA
[root@iceway-pc iceway]$ mount -t ext3 /dev/sdc2 /tmp/DATA/
[root@iceway-pc iceway]$ grub2-install --boot-directory=/tmp/DATA/ /dev/sdc
Installing for i386-pc platform.
Installation finished. No error reported.
[root@iceway-pc iceway]$ 
```

## 编写GRUB2配置文件

GRUB2现在已经被安装到U盘的第二个分区上，但是用该U盘启动会直接进入命令行模式，可以输入命令手动启动系统镜像或者加载windows的启动器。不过我们可以添加一个GRUB2配置文件，提供菜单选项启动系统镜像。配置文件位于boot分区的`/grub2/`，默认文件名为`grub.cfg`。

编辑文件`/tmp/DATA/grub2/grub.cfg`，内容如下（其中的uuid要替换成实际分区对应的uuid，可以通过命令`ls -l /dev/disk/by-uuid/`查看）：

```
menuentry "puppy linux 7.0.3 (april)" {
  echo "loading puppy 7.0.3 ..."
  insmod search_fs_uuid
  search --no-floppy --fs-uuid --set=root 9b6e6106-03b9-48ca-970d-39f67c2d1c1b
  set isofile="/puppy_7.0.3.iso"
  loopback loop $isofile
  linux (loop)/vmlinuz from=$isofile
  initrd (loop)/initrd.q
}

menuentry "openSUSE 42.1" {
  echo "loading openSUSE Leap 42.1"
  insmod ntfs
  insmod search_fs_uuid
  search --no-floppy --fs-uuid --set=root 64DCDFFA75D5B1E4
  set isofile="/openSUSE-Leap-42.1-DVD-x86_64.iso"
  loopback loop $isofile
  linux (loop)/boot/x86_64/loader/linux install=hd:$isofile exec="ln -s /usr/bin/mount /bin/mount"
  initrd (loop)/boot/x86_64/loader/initrd
}
```

>  我使用的完整配置文件参见：[grub.cfg](https://gist.github.com/iceway/3f8da875210219516c91b9b6be75a9ce)

## 拷贝ISO镜像

将openSUSE和Puppy的ISO镜像分别拷贝到U盘的第一个和第二个分区根目录下，命名必须和上一步配置文件中的文件名相同，注意区分大小写。
