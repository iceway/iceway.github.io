---
layout: post
title:  OpenWrt学习笔记（三）Feed
date:	23 Mar 2017 10:59:18 +0800
categories: Openwrt
tags: Openwrt
---

本文学习的内容可以参见[OpenWrt Feeds](https://wiki.openwrt.org/doc/devel/feeds)。

在OpenWrt中，一个“feed”就是一个位于同一位置的package的集合。Feeds可以位于一个远程服务器中，一个版本控制系统中，本地文件系统中，或者在一个可通过单一名称（一般而言是路径信息，如URL）由一种支持的feed方式定位到的其他任何地方。Feeds都是一些为OpenWrt Buildroot预先定义好的额外的packages的编译方法，OpenWrt项目默认自带了许多可用的package，用户可以自由的选择是否要编译某个package，如果你需要的某个package不是OpenWrt默认自带的，那就要自己手动添加相应的package。这时候Feeds就有用了，OpenWrt包含了很没在OpenWrt默认的packages中的包，如果需要的话可以从官方Feeds中下载这些包的packages以便在OpenWrt中配置并编译。当然，对于OpenWrt Feeds中也没有的包，也可以创建自己的Feeds添加这些包，后面将会专门说到如何创建OpenWrt的package。


## 使用Feeds

### Feed的配置

OpenWrt项目仓库根目录下，存在一个文件feeds.conf.default，内容如下：

```bash
src-git packages https://github.com/openwrt/packages.git;for-15.05
src-git luci https://github.com/openwrt/luci.git;for-15.05
src-git routing https://github.com/openwrt-routing/packages.git;for-15.05
src-git telephony https://github.com/openwrt/telephony.git;for-15.05
src-git management https://github.com/openwrt-management/packages.git;for-15.05
#src-git targets https://github.com/openwrt/targets.git
#src-git oldpackages http://git.openwrt.org/packages.git
#src-svn xwrt http://x-wrt.googlecode.com/svn/trunk/package
#src-svn phone svn://svn.openwrt.org/openwrt/feeds/phone
#src-svn efl svn://svn.openwrt.org/openwrt/feeds/efl
#src-svn xorg svn://svn.openwrt.org/openwrt/feeds/xorg
#src-svn desktop svn://svn.openwrt.org/openwrt/feeds/desktop
#src-svn xfce svn://svn.openwrt.org/openwrt/feeds/xfce
#src-svn lxde svn://svn.openwrt.org/openwrt/feeds/lxde
#src-link custom /usr/src/openwrt/custom-feed
```

这个文件包含了一系列feeds的列表，每行包含一个feed。以‘#’符号开始的行是注释行，在之后的feeds处理中会忽略这些注释行。每一个feed行由3个被空格分隔的部分组成，每部分分别是：feed方法，feed名称，feed源。

官网给出了OpenWrt feed可以支持的所有方法，如下表：

| 方法 | 功能解释 |
|-----|---------|
| src-bzr | 使用bzr通过feed源下载数据 |
| src-cpy | 从feed源直接拷贝数据 |
| src-darcs | 使用darcs通过feed源下载数据 |
| src-git | 使用git通过feed源下载数据，这也是OpenWrt现在默认所使用的方式 |
| src-gitsvn | 在subversion仓库和git仓库之间双向操作 |
| src-hg | 使用hg通过feed源下载数据 |
| src-link | 创建一个指向feed源的软连接 |
| src-svn | 使用svn通过feed源下载数据 |

feed方法的作用就在于让OpenWrt知道要通过那种方式或者协议获取对应feed源所指向的数据，现在git用的非常普遍，所以了解了src-git即可。可以看到src-git对应的feed源是一个git仓库的URL格式，可以通过git命令直接获取到这个仓库的数据。

feed名称用于标识feeds，并担当多个文件和目录名字的基础，被创建用于保存关于feeds的信息。而feed源就是指明了这个feed的数据要从哪里下载。

上述的feed方法中依赖版本控制系统的方法，支持一个“limited history”选项（就像git的`-depth`，和bzr的`-lightweight`），可以下载最小的有效历史。这是一个好的默认行为，但是那些向feed中活跃提交并使用提交历史的开发者或许想要改变这种行为。这个可以通过手动编辑文件`scripts/feeds`，或者不要使用`scripts/feeds`工具获取feed。

### Feed的命令

可以通过OpenWrt根目录下的`scripts/feeds`脚本文件使用feeds，不带参数执行该脚本可以的带所有可用的命令及参数。大多数的命令要求feed信息在本地可用，因此通常都需要先执行update命令将feeds从源更新的本地。`scripts/feeds`脚本用法如下：

```
Usage: ./scripts/feeds <command> [options]

Commands:
        list [options]: List feeds, their content and revisions (if installed)
        Options:
            -n :            List of feed names.
            -s :            List of feed names and their URL.
            -r <feedname>:  List packages of specified feed.
            -d <delimiter>: Use specified delimiter to distinguish rows (default: spaces)

        install [options] <package>: Install a package
        Options:
            -a :           Install all packages from all feeds or from the specified feed using the -p option.
            -p <feedname>: Prefer this feed when installing packages.
            -d <y|m|n>:    Set default for newly installed packages.
            -f :           Install will be forced even if the package exists in core OpenWrt (override)

        search [options] <substring>: Search for a package
        Options:
            -r <feedname>: Only search in this feed

        uninstall -a|<package>: Uninstall a package
        Options:
            -a :           Uninstalls all packages.

        update -a|<feedname(s)>: Update packages and lists of feeds in feeds.conf .
        Options:
            -a :           Update all feeds listed within feeds.conf. Otherwise the specified feeds will be updated.
            -i :           Recreate the index only. No feed update from repository is performed.

        clean:             Remove downloaded/generated files.
```

> 下面提到“合适的feeds”通常指在命令行在红给出的包名称或者当使用“-a”选项时表示所有的包。

#### clean

该命令删除本地保存的feed数据，包括feed中的所有包的索引和数据信息（但是不包括由install命令创建的软连接，该连接将指向空文件直到再次通过update命令重新下载这些feeds），该过程通过移除feeds的目录及其子目录完成。

#### install

安装适当的包及其依赖的任何包，安装过程包括创建从`packages/feeds/$feed_name/$package_name`到`feeds/$feed_name/$package_name`的软连接，以便这个包的目录层级被搜索时（OpenWrt配置时会自动搜索可用的所有包信息）在OpenWrt配置中可用。

| 命令 | 描述 |
|-----|------|
| ./scripts/feeds install -a | 安装所有包（不推荐这种用法，按需安装即可）|
| ./scripts/feeds install luci | 只安装LuCI包 |
| ./scripts/feeds install -a -p luci | 通过安装luci这个feed中的所有包来安装完整的LuCI WebUI部分 |

#### list

读取并显示在合适的feeds的索引中每一个feed包含的所有包，索引文件存储在feeds目录下，文件名是该feed名字加“.index”后缀。这些索引文件都是由update命令生成的。

#### search

搜索命令读取feed元数据并列出所有能匹配给定搜索条件的包。

#### uninstall

该命令的行为和install命令相反，但是并不会自动定位包依赖关系。举例讲，比如通过install命令安装coreutils包，根据依赖关系会自动安装acl和attr包，但是通过uninstall卸载coreutils包时不会自动分析并卸载acl和attr包，这点需要注意。卸载过程只是移除了install是创建的软连接。

#### update

该命令被执行时，每个合适的feeds被从其feed源位置下载到本地的一个名为feeds的目录下以其feed名称创建了的一个子目录中，然后分析这些feed中的包信息并生成索引文件，以便list和search命令使用。

值得注意的是update也会将feed的源路径信息存储到文件`feeds/$feed_name.tmp/location`中，这样的话对这些配置的更改会被检测到并适当的处理。

仅仅执行update命令，feed中的这些包还不能在OpenWrt配置界面中使用，还需要通过install命令安装才可以被配置接口识别并使用。

## 自定义Feeds

如果要使用官方的Feeds中没有的包，或者是自己创建的包，就可以自定义一个feed来管理这些包。两种方案：创建一个全新的feed，或者修改一个标准的feed。

### 创建包目录

第一种：将你的包添加到一个已存在的feed中（假设所有的操作都以“/home/user/openwrt”为根目录）

1. 创建你自己的项目目录`project`。
2. 然后进入`/home/user/openwrt/project`，并在该目录下操作。将OpenWrt trunk项目克隆到此目录下，命名为`openwrt`。将OpenWrt官方packages feed克隆到此目录下，命名为`packages`。
3. 将你自己的包添加到`/home/user/openwrt/project/packages`下的合适的子目录中。

第二种：创建自己的feed

1. 和上面的一样，先创建你的项目目录，并获取OpenWrt trunk。
2. 创建你自己的包目录，并将你的包拷贝至此。（比如：`cp packagedir /home/user/openwrt/project/customfeed/`），所以在这个例子这哦那个你的包位于`/home/user/openwrt/project/customfeed/packagedir`。

### 使用自定义的feed

上述第一种情况：

1. 编辑你的feeds.conf文件（如：/home/user/openwrt/project/openwrt/feeds.conf）。
2. 添加一个新行访问你定义的feed，同时将原本的packages feed注释掉。

```bash
#srv-svn packages svn://svn.openwrt.org/openwrt/packages
src-link customfeed /home/user/openwrt/project/packages
```

上述第二种情况：

直接添加新行定义你创建的feed，不用注释原本的。

```bash
src-link customfeed /home/user/openwrt/project/customfeed
```

接下来就可以使用自己定义的feed了。

1. 跟新feed，执行：
  ```
  ./scripts/feeds update customfeed
  ```
2. 安装：
  ```
  ./scripts/feeds install -p customfeed
  ```
3. 现在就可以在menuconfig中看到这些自定义包。

本文主要讲了如何使用及自己创建OpenWrt feed，并且知道feed可以包含很多额外package信息。那package到底是什么，都定义了哪些信息，下篇再了解pacakge。
