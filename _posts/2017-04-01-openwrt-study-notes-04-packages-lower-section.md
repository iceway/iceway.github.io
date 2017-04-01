---
layout: post
title:  OpenWrt学习笔记（四）Packages （下）
date:	01 Apr 2017 11:19:45 +0800
categories: Openwrt
tags: Openwrt
---

本文继续学习内容[Packages](https://wiki.openwrt.org/doc/devel/packages)。

## 配置一个软件包的源码

先看示例：

```makefile
CONFIGURE_ARGS += \
        --disable-native-affinity \
        --disable-unicode \
        --enable-hwloc

CONFIGURE_VARS += \
        ac_cv_file__proc_stat=yes \
        ac_cv_file__proc_meminfo=yes \
        ac_cv_func_malloc_0_nonnull=yes \
        ac_cv_func_realloc_0_nonnull=yes \
        CFLAGS="$(TARGET_CFLAGS) $(EXTRA_CFLAGS)" \
        CXXFLAGS="$(TARGET_CXXFLAGS) $(EXTRA_CFLAGS)" \
        CPPFLAGS="$(TARGET_CPPFLAGS) $(EXTRA_CPPFLAGS)" \
        LDFLAGS="$(TARGET_LDFLAGS) $(EXTRA_LDFLAGS)"
```

用于设置一些变量（autoconfig自己的变量或者CPPFLAGS，CFLAGS，CXXFLAGS，LDFLAGS等）或者configure的参数。设置configure的参数是很常见的，但是如果`configure.ac`和autoconf原始脚本在交叉编译中不能正常工作或无法发现某些库，就需要设置VARS。

### autotools：autoconf

针对autoconf可以配置的有：

* `CONFIGURE_VARS`： 覆盖`ac_cv_*`变量的值，这些变量一般在autoconf阶段设置。此外还会覆盖pkgconfi配置的变量。
* `CONFIGURE_ARGS`： 给要执行的`./configure`命令后面加一些参数。

### 编译器标志

默认的，在OpenWrt根目录的rule.mk文件中定义了以下这些有效的编译器标志

```makefile
TARGET_CFLAGS:=$(TARGET_OPTIMIZATION)$(if $(CONFIG_DEBUG), -g3) $(EXTRA_OPTIMIZATION)
TARGET_CXXFLAGS = $(TARGET_CFLAGS)
TARGET_ASFLAGS_DEFAULT = $(TARGET_CFLAGS)
TARGET_ASFLAGS = $(TARGET_ASFLAGS_DEFAULT)
TARGET_CPPFLAGS:=-I$(STAGING_DIR)/usr/include -I$(STAGING_DIR)/include
TARGET_LDFLAGS:=-L$(STAGING_DIR)/usr/lib -L$(STAGING_DIR)/lib
```

如果要改这些值，典型的做法是package makefile文件中为其追加内容， 如：

```makefile
TARGET_CFLAGS+= -Wall
```

### make

默认在OpenWrt根目录下文件include/package-defaults.mk中，定义了如下的make相关的标志：

```makefile
MAKE_VARS = \
	CFLAGS="$(TARGET_CFLAGS) $(EXTRA_CFLAGS) $(TARGET_CPPFLAGS) $(EXTRA_CPPFLAGS)" \
	CXXFLAGS="$(TARGET_CXXFLAGS) $(EXTRA_CXXFLAGS) $(TARGET_CPPFLAGS) $(EXTRA_CPPFLAGS)" \
	LDFLAGS="$(TARGET_LDFLAGS) $(EXTRA_LDFLAGS)"

MAKE_FLAGS = \
	$(TARGET_CONFIGURE_OPTS) \
	CROSS="$(TARGET_CROSS)" \
	ARCH="$(ARCH)"

MAKE_INSTALL_FLAGS = \
	$(MAKE_FLAGS) \
	DESTDIR="$(PKG_INSTALL_DIR)"

MAKE_PATH = .
```

如果要改变这些值，也是在package makefile中追加内容即可。

### cmake

同样的，在OpenWrt根目录下的include/cmake.mk文件中，定义了cmake相关的一些变量和标志，这里不再多说。

### 编译包需要主机上的工具

如果有些包的编译需要你的主机上安装的某些工具，而在OpenWrt的toolchain中没有，可以模仿下面这个片段告诉OpenWrt关于这些工具所在的更多信息。

```makefile
HOST_BUILD_DEPENDS:=<packagename>/host
PKG_BUILD_DEPENDS:=<packagename>/host
include $(INCLUDE_DIR)/host-build.mk

define Host/Compile
        XXX
endef

define Host/Install
        XXX
endef

$(eval $(call HostBuild))
```

### 注意

所有在pre/post install/removal脚本中的变量都应该用两个$$替代单个的$字符，这会告诉make工具不要把这个值解释为变量，而是仅仅忽略这个字符串并将其中的双$$替换成单个$。可以参见[这里](https://forum.openwrt.org/viewtopic.php?pid=85197#p85197)了解更多信息。

## 给package添加配置项（在menuconfig中可以操作的配置）

如果你想在menuconfig中提供接口配置你这个包的安装和编译过程，可以这样操作：

1: 给你的Package段定义中添加一个“MENU:=1”，如下：

```makefile
define Package/mjpg-streamer
  SECTION:=multimedia
  CATEGORY:=Multimedia
  TITLE:=MJPG-streamer
  DEPENDS:=@!LINUX_2_4 +libpthread-stubs +jpeg
  URL:=http://mjpg-streamer.wiki.sourceforge.net/
  MENU:=1
endef
```

2: 在package的Makefile文件中创建config关键字：

```makefile
define Package/mjpg-streamer/config
      source "$(SOURCE)/Config.in"
endef
```

3: 在这个Makefile所在的目录下创建一个名为“Config.in”的文件，内容如下：

```makefile
menu "Configuration"
	depends on PACKAGE_mjpg-streamer

config MJPEG_STREAMER_AUTOSTART
	bool "Autostart enabled"
	default n

	menu "Input plugins"
		depends on PACKAGE_mjpg-streamer
		config MJPEG_STREAMER_INPUT_FILE
			bool "File input plugin"
			help
				You can stream pictures from jpg files on the filesystem
			default n

		config MJPEG_STREAMER_INPUT_UVC
			bool "UVC input plugin"
			help
				You can stream pictures from an Universal Video Class compatible webcamera
			default y

		config MJPEG_STREAMER_FPS
			depends MJPEG_STREAMER_INPUT_UVC
			int "Maximum FPS"
			default 15

		config MJPEG_STREAMER_PICT_HEIGHT
			depends MJPEG_STREAMER_INPUT_UVC
			int "Picture height"
			default 640

		config MJPEG_STREAMER_PICT_WIDTH
			depends MJPEG_STREAMER_INPUT_UVC
			int "Picture width"
			default 480


		config MJPEG_STREAMER_DEVICE
			depends MJPEG_STREAMER_INPUT_UVC
			string "Device"
			default /dev/video0

		config MJPEG_STREAMER_INPUT_GSPCA
			bool "GSPCA input plugin"
			help
				You can stream pictures from a gspca supported webcamera Note this module is deprecated, use the UVVC plugin instead
			default n
	endmenu
endmenu
```

在这个示例中可以看到各种类型的配置参数，参考这个示例编辑你自己的配置文件即可。

接下来，你就可以在你的Makefile中检验你自定义的配置参数了（值得注意的是使用时你的配置参数名字前面应该要加上前缀`CONFIG_`，这个前缀是menuconfig配置中自动加上去的）。

```makefile
ifeq ($(CONFIG_MJPEG_STREAMER_INPUT_UVC),y)
	$(CP) $(PKG_BUILD_DIR)/input_uvc.so $(1)/usr/lib
endif
```

## 为内核模块创建package

OpenWrt中的内核模块也位于package目录下，但是内核模块的Makefile和普通包的Makefile有些区别。

很多的内核程序都在含在内核的源代码分发包中，构建内核时可以选择将这些内核程序：

1. 编译到内核镜像中作为一个整体
2. 编译成一个可加载的内核模块
3. 忽略该内核程序，不要编译

在OpenWrt中，为了包含这些内核程序（作为可加载模块），直接在menuconfig中选择相关的内核选项即可。OpenWrt已经将一些常见的在内核源代码中的内核模块的选项加入到menuconfig的配置项中。如果你需要的内核模块没有出现在OpenWrt的配置菜单中，你必须在目录package/kernel/linux/modules下的某个文件中手动添加一节内容。下面给出一个示例（取自package/kernel/linux/modules/block.mk）：

```makefile
define KernelPackage/loop
  SUBMENU:=$(BLOCK_MENU)
  TITLE:=Loopback device support
  KCONFIG:= \
        CONFIG_BLK_DEV_LOOP \
        CONFIG_BLK_DEV_CRYPTOLOOP=n
  FILES:=$(LINUX_DIR)/drivers/block/loop.ko
  AUTOLOAD:=$(call AutoLoad,30,loop)
endef

define KernelPackage/loop/description
 Kernel module for loopback device support
endef

$(eval $(call KernelPackage,loop))
```

对这些mk文件的修改，不会被OpenWrt的构建系统自动收集到，也就是说即使修改了这些文件并保存后，并不能立即在menuconfig中看到这些改动的内容。要想强制OpenWrt重新读取这些改动有两种方法：一是通过命令`touch package/kernel/linux/Makefile`更新文件package/kernel/linux/Makefile的时间戳；另一个是删除编译根目录下的tmp目录。

注意，上面说到的关于内核的配置项，是针对那些源代码就在内核源代码目录下的内核程序而言。如果你的内核模块源代码不属于Linux内核源码的一部分（也就是说你的内核模块代码在Linux内核源码目录之外），这种情况下就需要在package目录下创建一个内核模块的目录，类似普通的包，只是其中的Makefile文件需要注意：要用*`KernelPackage/xxx`*代替*`Package/xxx`*

下面是package/madwifi/Makefile的内容，作为参考：

```makefile
#
# Copyright (C) 2006 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
# $Id$

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=madwifi
PKG_VERSION:=0.9.2
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.bz2
PKG_SOURCE_URL:=@SF/$(PKG_NAME)
PKG_MD5SUM:=a75baacbe07085ddc5cb28e1fb43edbb
PKG_CAT:=bzcat

PKG_BUILD_DIR:=$(KERNEL_BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

RATE_CONTROL:=sample

ifeq ($(ARCH),mips)
  HAL_TARGET:=mips-be-elf
endif
ifeq ($(ARCH),mipsel)
  HAL_TARGET:=mips-le-elf
endif
ifeq ($(ARCH),i386)
  HAL_TARGET:=i386-elf
endif
ifeq ($(ARCH),armeb)
  HAL_TARGET:=xscale-be-elf
endif
ifeq ($(ARCH),powerpc)
  HAL_TARGET:=powerpc-be-elf
endif

BUS:=PCI
ifneq ($(CONFIG_LINUX_2_4_AR531X),)
  BUS:=AHB
endif
ifneq ($(CONFIG_LINUX_2_6_ARUBA),)
  BUS:=PCI AHB	# no suitable HAL for AHB yet.
endif

BUS_MODULES:=
ifeq ($(findstring AHB,$(BUS)),AHB)
  BUS_MODULES+=$(PKG_BUILD_DIR)/ath/ath_ahb.$(LINUX_KMOD_SUFFIX)
endif
ifeq ($(findstring PCI,$(BUS)),PCI)
  BUS_MODULES+=$(PKG_BUILD_DIR)/ath/ath_pci.$(LINUX_KMOD_SUFFIX)
endif

MADWIFI_AUTOLOAD:= \
	wlan \
	wlan_scan_ap \
	wlan_scan_sta \
	ath_hal \
	ath_rate_$(RATE_CONTROL) \
	wlan_acl \
	wlan_ccmp \
	wlan_tkip \
	wlan_wep \
	wlan_xauth

ifeq ($(findstring AHB,$(BUS)),AHB)
	MADWIFI_AUTOLOAD += ath_ahb
endif
ifeq ($(findstring PCI,$(BUS)),PCI)
	MADWIFI_AUTOLOAD += ath_pci
endif

define KernelPackage/madwifi
  SUBMENU:=Wireless Drivers
  DEFAULT:=y if LINUX_2_6_BRCM |  LINUX_2_6_ARUBA |  LINUX_2_4_AR531X |  LINUX_2_6_XSCALE, m if ALL
  TITLE:=Driver for Atheros wireless chipsets
  DESCRIPTION:=\
	This package contains a driver for Atheros 802.11a/b/g chipsets.
  URL:=http://madwifi.org/
  VERSION:=$(LINUX_VERSION)+$(PKG_VERSION)-$(BOARD)-$(PKG_RELEASE)
  FILES:= \
		$(PKG_BUILD_DIR)/ath/ath_hal.$(LINUX_KMOD_SUFFIX) \
		$(BUS_MODULES) \
		$(PKG_BUILD_DIR)/ath_rate/$(RATE_CONTROL)/ath_rate_$(RATE_CONTROL).$(LINUX_KMOD_SUFFIX) \
		$(PKG_BUILD_DIR)/net80211/wlan*.$(LINUX_KMOD_SUFFIX)
  AUTOLOAD:=$(call AutoLoad,50,$(MADWIFI_AUTOLOAD))
endef

MADWIFI_MAKEOPTS= -C $(PKG_BUILD_DIR) \
		PATH="$(TARGET_PATH)" \
		ARCH="$(LINUX_KARCH)" \
		CROSS_COMPILE="$(TARGET_CROSS)" \
		TARGET="$(HAL_TARGET)" \
		TOOLPREFIX="$(KERNEL_CROSS)" \
		TOOLPATH="$(KERNEL_CROSS)" \
		KERNELPATH="$(LINUX_DIR)" \
		LDOPTS=" " \
		ATH_RATE="ath_rate/$(RATE_CONTROL)" \
		DOMULTI=1

ifeq ($(findstring AHB,$(BUS)),AHB)
  define Build/Compile/ahb
	$(MAKE) $(MADWIFI_MAKEOPTS) BUS="AHB" all
  endef
endif

ifeq ($(findstring PCI,$(BUS)),PCI)
  define Build/Compile/pci
	$(MAKE) $(MADWIFI_MAKEOPTS) BUS="PCI" all
  endef
endif

define Build/Compile
	$(call Build/Compile/ahb)
	$(call Build/Compile/pci)
endef

define Build/InstallDev
	$(INSTALL_DIR) $(STAGING_DIR)/usr/include/madwifi
	$(CP) $(PKG_BUILD_DIR)/include $(STAGING_DIR)/usr/include/madwifi/
	$(INSTALL_DIR) $(STAGING_DIR)/usr/include/madwifi/net80211
	$(CP) $(PKG_BUILD_DIR)/net80211/*.h $(STAGING_DIR)/usr/include/madwifi/net80211/
endef

define KernelPackage/madwifi/install
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_DIR) $(1)/lib/modules/$(LINUX_VERSION)
	$(INSTALL_DIR) $(1)/usr/sbin
	$(INSTALL_BIN) ./files/madwifi.init $(1)/etc/init.d/madwifi
	$(CP) $(PKG_BUILD_DIR)/tools/{madwifi_multi,80211debug,80211stats,athchans,athctrl,athdebug,athkey,athstats,wlanconfig} $(1)/usr/sbin/
endef

$(eval $(call KernelPackage,madwifi))
```

一些相关宏在文件include/kernel.mk中定义，具体可以阅读这个文件。

## 安装文件的相关宏

在OpenWrt根目录下的rules.mk文件中，定义了下面一些宏，可以用于在Package/XXX/install中安装文件：

```makeifle
INSTALL_BIN:=install -m0755
INSTALL_DIR:=install -d -m0755
INSTALL_DATA:=install -m0644
INSTALL_CONF:=install -m0600

CP:=cp -fpR
LN:=ln -sf
```
