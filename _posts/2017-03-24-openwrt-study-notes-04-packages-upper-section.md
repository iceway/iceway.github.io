---
layout: post
title:  OpenWrt学习笔记（四）Packages （上）
date:	24 Mar 2017 16:53:05 +0800
categories: Openwrt
tags: Openwrt
---

本文学习内容参见[Packages](https://wiki.openwrt.org/doc/devel/packages)。

OpenWrt构建系统可以让用户自由选择并配置大量不同的软件包，每个软件包基本上就是一个package，只是OpenWrt中每个package要有特定的格式以便OpenWrt可以识别。下面通过创建pacakge的过程学习package的具体格式。

OpenWrt根目录中的子目录`package`中包含了许多独立的子目录（现在的OpenWrt在package目录下多了一层或多层分类目录，实际的package目录位于这些分类目录中），每个子目录表示了一个对应的package。比如，在OpenWrt 15.05版中查看目录“package/utils/”，可以看到列出的这些目录每一个都表示一个package。之前介绍过的Feed实际上就是大量package的合集，用于给OpenWrt默认自带的package增加新的内容。

```bash
openwrt.git git:(chaos_calmer) ls package/utils
admswconfig  bzip2      fbtest  hostap-utils  lua    mkelfimage  otrx  px5g-standalone  spidev_test  ugps     usbreset  util-linux
busybox      e2fsprogs  fuse    jsonfilter    mdadm  nvram       px5g  robocfg          ubi-utils    usbmode  usbutils  xfsprogs
```

一个典型的package中包含以下三项：

1. package/Makefile
2. package/patches/
3. package/files/

其中，Makefile文件是重点，它提供了实际下载并编译该package所需的详细步骤。

patches目录是可选的，其中一般包含了一些针对软件官方原始代码的bug fixes，以及优化内容以减小可执行文件大小，或者针对OpenWrt系统编译构建过程的适配等。OpenWrt的很多软件包都是开源的第三方软件，有时OpenWrt开发组发现一些软件的bug，但是不能及时合并到这些上游软件的官方release中，就会在OpenWrt中通过上游软件官方源码包加上OpenWrt自己开发的本地补丁解决，这些本地补丁就存放在各个package的patches目录中。当然，也包含一些针对OpenWrt系统优化，或者编译过程等的适配修改也在这个目录下以补丁形式存在。可以这么理解，OpenWrt中如果某个软件包的官方发布的源码包不能满足OpenWrt的需求，就会在其patches目录下添加一些补丁修改。

files目录也是可选的，一般在这个目录下包含一些针对该软件的默认配置及启动脚本等文件。

OpenWrt为了让一个软件可以很容易就导入到OpenWrt构建系统中，编写了OpenWrt模板系统，而package的定义就需要满足特定的模板格式，接下来看一个具体的例子来了解更多信息，很难看出这是一个makefile文件，通过这些只能被我们描述为公然无视及滥用传统的make格式的内容，makefile已经被改成一个面向对象的模板并简化了操作的难度。

下面是一个示例文件的内容，文件是“package/feeds/packages/bridge-utils/Makefile”，接下来我们逐步讲解这类makefile中的规则及格式。

```makefile
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=bridge-utils
PKG_VERSION:=1.5
PKG_RELEASE:=3

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://git.kernel.org/pub/scm/linux/kernel/git/shemminger/bridge-utils.git
PKG_SOURCE_VERSION:=v${PKG_VERSION}
PKG_SOURCE_SUBDIR:=$(PKG_NAME)-$(PKG_VERSION)
PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz

PKG_LICENSE:=GPL-2.0+
PKG_LICENSE_FILES:=COPYING
PKG_FIXUP:=autoreconf

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz

include $(INCLUDE_DIR)/package.mk

define Package/bridge
  SECTION:=net
  CATEGORY:=Base system
  TITLE:=Ethernet bridging configuration utility
  URL:=http://bridge.sourceforge.net/
  PKG_MAINTAINER:=Nikolay Martynov <mar.kolya@gmail.com>
endef

define Package/bridge/description
 Manage ethernet bridging: a way to connect networks together to
 form a larger network.
endef

TARGET_CFLAGS += -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE

CONFIGURE_ARGS += \
        --with-linux-headers="$(LINUX_DIR)" \

define Package/bridge/install
        $(INSTALL_DIR) $(1)/usr/sbin
        $(INSTALL_BIN) $(PKG_BUILD_DIR)/brctl/brctl $(1)/usr/sbin
endef

$(eval $(call BuildPackage,bridge))
```

## 构建包的变量

如你所见，makefile中并没有太多工作可做，所有的工作都被隐藏在其他的makfile中，并且抽象出一些要点，你只需要执行少量的变量。

* `PKG_NAME` - 定义这个包的名字，在menucongfig中看到的包名字就是这里指定的。
* `PKG_VERSION` - 定义了我们要下载的软件的上游版本号。
* `PKG_RELEASE` - 这个包的Makefile版本号，可以理解为OpenWrt自己对每个pacakge赋予的版本。
* `PKG_LICENSE` - 这个包的授权协议，一般都来自上游软件商。
* `PKG_LICENSE_FILE` - 描述授权协议的文件。
* `PKG_BUILD_DIR` - 将要在那个位置存储并编译这个包的软件源码。
* `PKG_SOURCE` - 这个包的原始码的文件名。
* `PKG_SOURCE_URL` - 从哪里下载这个包的源码。
* `PKG_MD5SUM` - 用于验证所下载内容的校验码，可以使MD5或者SHA256其中之一。
* `PKG_CAT` - 定义要用哪种方式压缩并释放原始码（zcat，bzcat，还是unzip）
* `PKG_BUILD_DEPENDS` - 定义这要编译这个包之前必须先编译的包，也就是这个包在编译过程的依赖包，运行时不一定需要。一般而言都是一些提供链接时的依赖库或者头文件等的包，使用的语法和后面要说到的DEPENDS相同。
* `PKG_INSTALL` - 如果将这个变量设置为1，将会调用这个软件原代码中的原生`make install`，并将prefix设置为`PKG_INSTALL_DIR`的值。
* `PKG_INSTALL_DIR` - 定义了`make install`将会把编译好的文件拷贝到哪里。
* `PKG_FIXUP` - 这个变量的含义后面会详述，请参见下面的内容。
* `PKG_SOURCE_PROTO` - 定义了用于获取原始代码的协议（git，svn）
* `PKG_REV` - svn revision要用的到，如果proto是svn时必须指定。
* `PKG_SOURCE_SUBDIR` - 如果proto是svn或者git必须要指定这个，如：`PKG_SOURCE_SUBDIR:=$(PKG_NAME)-$(PKG_VERSION)`
* `PKG_SOURCE_VERSION` - 如果proto是git必须指定这个，表示git中的commit hash（一个tag名或者commit id），用于git的checkout操作。
* `PKG_CONFIG_DEPENDS` - 指定了哪些依赖这个包的配置选项将被选中。

这里面的变量并不是所有的都需要定义，如果需要自己创建package，可以参考OpenWrt已有的包的Makefile去写。

在上面给出的示例makefile文件最底部的那行调用BuildPackage的语句才是真正魔法发生的所在，也是make真正开始执行命令的开始。“BuildPackage”是一个在前面include包含的文件中声明的宏（在make中宏可以理解为C中的函数，可以给宏传递参数并调用），该宏仅直接接受一个参数 - 就是要编译的这个包的名字，在这个示例中是bridge。所有其他的信息从define块中得到。

为保持定义和重计算这行的一致性和可读性，要避免在最后的call BuildPackage语句中使用`PKG_NAME`变量，直接使用完整名字替代。

### PKG_FIXUP

许多软件包认为autotools是一个好工具，最终需要修复来解决自动化工具“意外的”认为使用主机工具（就是你的电脑系统上安装的一些构建工具）替代构建环境的工具（指OpenWrt项目上的toolchain中提供的构建工具）更好。OpenWrt定义一些`PKG_FIXUP`规则帮助解决此事。

`PKG_FIXUP`可以是下面几个值之一：

1. autoreconf：这将会执行以下动作
  * autoreconf -f -i
  * 创建必须但是缺失的文件
  * 保证openwrt-libtool已被链接好
  * 阻止autopoint/gettext
2. patch-libtool： 如果释出的automake方法已破坏且无法修复，则寻找一个libtool实例，检测其版本并应用OpenWrt的修正补丁。
3. gettext-version： 这个fixup在automake的gettext支持中阻止了版本不匹配的错误。

> 提醒：使用autotool工具的软件包应该通过简单的指定`PKG_FIXUP:=autoreconf`正常工作，否则就可能出现要求特定版本的问题。

## 软件包的源码

OpenWrt构建系统支持多种不同方式下载外部源码包。

### 使用打包过的源码包

大多数软件包使用一个“.tar.gz”，“.tar.bz2”，或者类似格式打包其源码文件。

### 代码仓库
通过变量`PKG_SOURCE_PROTO`可以支持从多种仓库下载源码并整合开发版本。

```makefile
PKG_SOURCE_PROTO:=bzr
PKG_SOURCE_PROTO:=cvs
PKG_SOURCE_PROTO:=darcs
PKG_SOURCE_PROTO:=git
PKG_SOURCE_PROTO:=hg
PKG_SOURCE_PROTO:=svn
```

### 随OpenWrt Makefile文件一起捆绑源代码

OpenWrt中也可以直接将软件包的源代码放在`package/<packagename>`目录下，一般保存在一个名为src的子目录中。如包px5g-standalone的pacakge文件结构：

```
openwrt.git git:(chaos_calmer) ls -R package/utils/px5g-standalone  
package/utils/px5g-standalone:
Makefile  src

package/utils/px5g-standalone/src:
library  Makefile  polarssl  px5g.c

package/utils/px5g-standalone/src/library:
base64.c  bignum.c  havege.c  rsa.c  sha1.c  timing.c  x509write.c

package/utils/px5g-standalone/src/polarssl:
base64.h  bignum.h  bn_mul.h  config.h  havege.h  rsa.h  sha1.h  timing.h  x509.h
```

### 下载

捆绑在OpenWrt中的源代码不需要覆盖。

可以从外部源下载额外数据，如下面的示例。一般使用Download都是下载额外的数据，如果是这个package本身的源码，通过前面定义的变量所获取的信息，OpenWrt会自动下载对应的源码包。

```makefile
USB_IDS_VERSION:=2013-01-16
USB_IDS_MD5SUM:=2a2344907b6344f0935c86efaf9de620
USB_IDS_FILE:=usb.ids.$(USB_IDS_VERSION).gz

[...parts missing...]

define Download/usb_ids
  FILE:=$(USB_IDS_FILE)
  URL:=http://mirror2.openwrt.org/sources
  MD5SUM:=$(USB_IDS_MD5SUM)
endef
$(eval $(call Download,usb_ids))
```

然后解包这个文件，或者将其整合到构建过程中：

```makefile
define Build/Prepare
        $(Build/Prepare/Default)
        echo '#!/bin/sh' > $(PKG_BUILD_DIR)/update-usbids.sh.in
        echo 'cp $(DL_DIR)/$(USB_IDS_FILE) usb.ids.gz' >> $(PKG_BUILD_DIR)/update-usbids.sh.in
endef
```

也可以在“Build/Prepare”节中修改变量`UNPACK_CMD`或者手动调用/修改`PKG_UNPACK`，没有修改`PKG_UNPACK`的话，执行`PKG_UNPACK`后实际上还是调用`UNPACK_CMD`。

```makefile
UNPACK_CMD=ar -p "$(DL_DIR)/$(PKG_SOURCE)" data.tar.xz | xzcat | tar -C $(1) -xf -
define Build/Prepare
        $(PKG_UNPACK)
#       we have to download additional stuff before patching
        (cd $(PKG_BUILD_DIR) && ./contrib/download_prerequisites)
        $(Build/Patch)
endef
```

## Package的定义

Package的Makefile中可以使用define关键字定义很多section，下面就介绍常见的各种section的作用。

### Package/{package-name}

匹配的是传递给buildroot的参数，这段描述的是这个package在menuconfig及ipkg中的入口。可以定义的变量有：

* SECTION：这个包的类型（目前暂时没有用）。
* CATEGORY：定义了在menuconfig中这个包要出现在那个子目录（菜单）下，就是menuconfig中的菜单类别。
* TITLE：针对这个包的一个简短描述。
* DESCRIPTION：具体描述这个包（现在已不赞成使用这个变量描述包，而是用下面提到的Package/description段替代）。
* URL：这个软件包的官方网站。
* MAINTAINER：（少量包需要这个变量）指出有这个包的官方维护者，也就是对这个包有任何问题时应该联系谁。
* DEPENDS：（可选）定义了这个包编译之前要先编译并安装哪些包，也就是这个包的依赖包信息，后面再专门解释依赖的语法。
* PKGARCH：（可选）这个包的架构平台，设置为“all”可以产生一个“Architecture: all”的包。
* USERID：（可选）在这个包安装时的一个“username:groupname”对。

### Package/{package-name}/conffiles （可选）

一份由这个包安装的配置文件的列表，每行一个文件。

### Package/{package-name}/description

任何用来描述这个包的文字。

### Build/Prepare （可选）

一些命令的集合，用于解包源码以及应用补丁，可以不定义这个段，OpenWrt会自动检查源码包的格式并做出适当处理，并应用所有存在的本地补丁。

这个命令的作用就是在编译一个软件包之前，先将这个软件包编译需要的所有源码都准备好。

### Build/Configure （可选）

很多开源软件的源码在开始编译前要先执行`./configure XXX`类似的命令做一些配置及检查工作，这个段的作用就是描述如何配置这个包的。如果你的源码不需要配置或者就是常规的配置脚本（configure等），就不用定义这个段，否则你要在这个段中定义适合你的包的命令来配置代码，或者使用`$(call Build/Configure/Default,)`语句并将所需的其他参数传递给标准的配置脚本。

### Build/Compile （可选）

定义如何去编译这个包的源码，大多数情况下不用定义这个段，因为OpenWrt会自动执行默认动作，执行make。如果你想传递给make一些特殊参数，可以像这个调用方式一样操作：`$(call Build/Compile/Default,FOO=bar)`。

### Build/Install （可选）

定义如何安装编译好的代码，默认动作是执行`make install`。和上面的类似，如果想传递特殊参数，使用`$(call Build/Install/Default,install install-foo)`。注意你必须将所有需要的参数都在此指定，如果你只想给install参数添加一些内容，别忘了还要传递install本身。实际上就是说可以在此指定多个make目标，像示例的代码则会执行`make install install-foo`。

### Build/InstallDev （可选）

处理一些OpenWrt编译包时可以依赖的文件（如静态库，头文件等），但是这些在目标设备上用不到。举例来说，假设你的OpenWrt项目上有一个基本的包，这个包中的一些头文件在编译其他包时会用到，但是最终生成的固件镜像烧入目标设备后却用不到这些头文件，就可以在这个section中定义要将这些头文件拷贝到哪里去（一般是toolchain使用的头文件路径）。当然如果其他包编译时需要用到这个包的头文件，那么其他包也应该定义为依赖这个包，这样在其他包编译之前会先编译这个包，并执行这些install动作以免其他包编译时找不到头文件。

### Package/{package-name}/install

一系列的命令集合，用于将文件拷贝到ipkg目录（用$(1)表示）。OpenWrt把每个包在各自的源码目录编译好后，安装过程会在这个包的编译目录下创建一个ipkg目录（名字有可能会是`ipkg-<target>`形式，target表示OpenWrt menuconfig中设置的target），然后将所有需要包含到最终文件系统的文件按照其完整路径拷贝到这个ipkg目录中，最后在生成文件系统时会把所有的ipkg目录中的内容都复制到文件系统目录下，然后才生成最终的固件。

文件要安装的目标目录肯定是ipkg目录（用$(1)表示），而源路径可以用相对路径表示buildroot/pacakge/{package-name}，也可以用`$(PKG_INSTALL_DIR)`表示Build/Install中文件所在目录，`$(PKG_BUILD_DIR)`表示这个包的实际代码全部解包所至的目录并且编译好的文件一般也位于这个目录。

看看package/devel/gdb/Makefile中的这段makefile可以比较清楚的理解Build/Install和Package/{package-name}/install的差别：

```makefile
define Build/Install
        $(MAKE) -C $(PKG_BUILD_DIR) \
                DESTDIR="$(PKG_INSTALL_DIR)" \
                CPPFLAGS="$(TARGET_CPPFLAGS)" \
                install-gdb
endef

define Package/gdb/install
        $(INSTALL_DIR) $(1)/usr/bin
        $(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/gdb $(1)/usr/bin/
endef
```

### Package/{package-name}/preinst

一个在安装前执行的脚本的实际内容，不要忘记了包含`#!/bin/sh`，如果你想终止安装，只要让这个脚本直接返回false。

### Package/{package-name}/postinst

一个在安装后执行的脚本的实力内容，不要忘记了包含`#!/bin/sh`。

### Package/{package-name}/prerm

一个在删除前执行的脚本的实际内容，不要忘记了包含`#!/bin/sh`，如果你想终止删除，只要让这个脚本直接返回false。

### Package/{package-name}/postrm

一个在删除后执行的脚本的实际内容，不要忘记了包含`#!/bin/sh`。

为什么有些define以“Package/”作为前缀，而其他的已“Build/”作为前缀的原因是为了可以从单一的源处生成多个package。OpenWrt的工作假设每一个源都对应一个package makefile，但是你可以将这个源分割成许多你想要的package，也就是说对于一个package目录，你可以在其Makefile中定义多个Package，这些不同的Package在menuconfig中是独立的，看起来好像有很多个独立的pacakge目录和makefile。对于这样的包，由于你只需要把源码编译一次（可以定义不同的pacakge安装不同的编译好的文件，如某个软件包中包含了server和client的源码，可以分别定义一个server和client的package，安装不同的package时要复制不同的文件，但是这个软件包只需要一次编译就可以将server和client的部分都编译好），所以有一个总的“Build/”集合，但是你可以定义任意多你想要的“Package/”，并且通过添加额外的BuildPacakge调用实现多个分割的package。比如，“package/devel/gdb/Makefile”文件最后的内容：

```makefile
$(eval $(call BuildPackage,gdb))
$(eval $(call BuildPackage,gdbserver))
```

可以看到，所有名字前缀是“Build/”的section都是直接定义，而名字前缀是“Package/”的section都需要在名字中间加上具体的package名字。这也说明了Build是全局的，而Package是针对每个package定义的。

## 在源码目录的子目录中构建

有些软件包源码以多个不同目录管理软件内容，比如源代码和Makefile都放在src子目录下，而文档全部放在doc子目录下等。问题在于OpenWrt构建系统将会尝试在`$(PKG_BUILD_DIR)`目录下执行make，但是由于Makefile在其子目录src下导致make失败。要解决这类问题，可以通过变量`MAKE_PATH`指定具体的子目录，比如：

```makefile
MAKE_PATH:=src
```

这个变量所制定的路径会以`$(PKG_BUILD_DIR)`作为相对路径，如果没有指定时默认值是`.`。

## “依赖”的具体语法

可以指定多种不同类型的依赖，不过需要一点解释来了解其中的不同，关于依赖的更多文档可以参见[Using Dependencies](https://wiki.openwrt.org/doc/devel/dependencies)。

### `+<foo>`

这种依赖格式表示当前的pacakge依赖`<foo>`这个pacakge，并且如果在menuconfig中选中了当前的pacakge，就会自动选中依赖包`<foo>`。

### `<foo>`

表示当前的package依赖`<foo>`这个pacakge，并且只有当依赖包`<foo>`被选中时才可以在menuconfig中看到当前包。

### `@FOO`

表示当前包依赖配置文件中的配置符号`CONFIG_FOO`，并且只有当`CONFIG_FOO`被选中时才可以在menuconfig中看到当前包。这种格式一般用在依赖一个确定的Linux版本或者目标时，比如依赖值是`@TARGET_foo`会让一个包只在foo这个目标下有效。可使用布尔表达式表示复杂的依赖关系，比如：`@(!TARGET_foo&&!TARGET_bar)`让这个包对foo和bar两个目标无效，也就是说当目标是这两者之一时，当前的pacakge在menuconfig中不可见。

### `+FOO:<bar>`

表示当前包在`CONFIG_FOO`被设置的情况下依赖`<bar>`这个包，并且选中当前包会自动选择`<bar>`这个包。这种格式的典型用法是如果这个包的编译时选项会触发其他依赖外部库的特性时使用。

### `@FOO:<bar>`

表示当前包在`CONFIG_FOO`被设置的情况下依赖`<bar>`这个包，并且在`CONFIG_FOO`被设置且`<bar>`包被选择之前在menuconfig中不可见。
