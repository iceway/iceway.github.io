---
layout: post
title:  OpenWrt学习笔记（七）系统初始化脚本
date:	14 Apr 2017 17:00:26 +0800
categories: Openwrt
tags: Openwrt
---

本文学习内容[SysV style init scripts](https://wiki.openwrt.org/doc/techref/initscripts)，[procd init scripts](https://wiki.openwrt.org/inbox/procd-init-scripts)，[procd](https://wiki.openwrt.org/doc/techref/procd)。

## SysV风格的init脚本

在最新的OpenWrt中，已经用procd取代SysV风格的init脚本，本文后面会再介绍procd。

init脚本设置Linux系统的守护进程。init脚本作为boot进程的一部分被执行用来启动其他一些必须的进程。在OpenWrt以前的版本中，init脚本是和init.d一起实现的。在启动过程中调用这些启动脚本的init进程是在buxybox中实现的，下面解释了init.d及脚本是如何工作的，以及如何创建这些脚本。

下面用一个示例说明基本的init脚本，假设我们有一个守护进程想让init.d处理，首先创建一个文件`/etc/init.d/example`（文件名随意取，表明脚本的用途即可），最简单的内容看起来如下：

```bash
#!/bin/sh /etc/rc.common
# Example script
# Copyright (C) 2007 OpenWrt.org

START=10
STOP=15

start() {        
        echo start
        # commands to launch application
}                 

stop() {          
        echo stop
        # commands to kill application
}
```

init脚本实际上就是一个shell脚本，第一行是一个[shebang line](https://en.wikipedia.org/wiki/Shebang_(Unix))，它使用/etc/rc.common提供了塔的主要默认功能函数，并且在执行之前检查这个脚本。这句话说的有点拗口，简单的理解就是第一行的作用在于，如果执行命令`/etc/init.d/example start`，实际上真正执行的命令是`/bin/sh /etc/rc.common /etc/init.d/example start`，查看rc.common文件的内容可以更好的理解其作用。rc.common脚本的内容比较简单，这里不再多说，默认情况下，rc.common提供了一些命令模板如下：

```
        start   Start the service
        stop    Stop the service
        restart Restart the service
        reload  Reload configuration files (or restart if that fails)
        enable  Enable service autostart
        disable Disable service autostart
```

所有这些列出的命令都可以在执行时传递给init脚本，比如重启example服务可以以restart参数调用这个脚本：

```bash
/etc/init.d/example restart
```

init脚本中一般只需要自己定义start和stop两个函数，通过这两个函数决定在启动或停止该服务时所需的核心步骤（也就是具体要启动或者杀死的程序，及其配置等）。

示例中的`START=`和`STOP=`表明在系统初始化顺序中的哪个点执行这个脚本。启动时，init.d之根据/etc/rc.d目录下发现的脚本的文件名依次启动这些脚本。通过在/ect/rc.d目录下创建一个链接到/etc/init.d目录下某个文件的软链接让启动过程自动执行这个脚本，这些软链接的文件名要求是形如`S##XXXXX`或者`K##XXXXXX`（其中##表示两位数字，XXXXXX表示/etc/init.d目录下的原始文件名，S和K的区别在上一篇介绍启动过程时说过了。）。通过enable和disable命令可以自动创建或者删除这些软链接，在创建过程中START和STOP的值就会被用到，START表示创建的是`S##XXXXXX`形式的软链接，而STOP表示创建的是`K##XXXXXX`形式的软链接。比如，这个示例中：

* `START=10`表示enable命令会自动创建软链接`/etc/rc.d/S10example`。
* `STOP=15`表示enable命令会自动创建软连接`/etc/rc.d/K15example`。

> 注意：别忘记给自己创建的init脚本文件添加可执行权限。

boot命令只在系统启动时执行一次，如果你的init文件有一些动作只是在板子启动时要执行一次（参见上一篇介绍启动过程的文章），而后续重启该init服务时并不需要重新执行的动作，可以写在boot函数中，rc.common中默认定义的boot函数如下，如果没有自己定义boot函数，则执行start命令。

```bash
boot() {
        start "$@"
}
```

另外，你还可以添加自定义的命令。使用`EXTRA_COMMANDS`和`EXTRA_HELP`变量分别定义你的自定义命令的名称和帮助文档，然后再添加一个同名函数即可，示例如下：

```bash
EXTRA_COMMANDS="custom"
EXTRA_HELP="        custom  Help for the custom command"

custom() {
        echo "custom command"
        # do your custom stuff
}
```

添加这个命令后，example脚本的可接受参数如下：

```
root@OpenWrt:/# /etc/init.d/example
Syntax: /etc/init.d/example [command]

Available commands:
        start   Start the service
        stop    Stop the service
        restart Restart the service
        reload  Reload configuration files (or restart if that fails)
        enable  Enable service autostart  
        disable Disable service autostart
        custom  Help for the custom command
```

添加多个自定义命令的示例如下：

```bash
EXTRA_COMMANDS="custom1 custom2 custom3"
EXTRA_HELP=<<EOF
        custom1 Help for the custom1 command
        custom2 Help for the custom2 command
        custom3 Help for the custom3 command
EOF

custom1 () {
        echo "custom1"
        # do the stuff for custom1
}
custom2 () {
        echo "custom2"
        # do the stuff for custom2
}
custom3 () {
        echo "custom3"
        # do the stuff for custom3
}
```

注意：很多有用的守护进程都已被包含的正式发布的OpenWrt镜像中，但是有些init脚本默认没有enable（也就是没有在/etc/rc.d下创建软链接）。比如cron守护进程默认没有激活，如果只是修改crontab实际上并不会生效，必须手动启动（start）该守护进程或者激活（enable）。

下面的命令可以查询所有init脚本是否已被激活（enabled）：

```bash
for F in /etc/init.d/* ; do $F enabled && echo $F on || echo $F **disabled**; done
```

## procd init脚本

### 什么是procd

procd是OpenWrt使用的新的进程管理服务，用C语言编写。它持续追踪从init脚本中启动的进程（通过ubus调用），并且可以在配置/环境没有更改时避免无谓的服务重启请求。就是说，如果你用procd管理你的所有init启动的服务进程，如果下次又收到重启该init服务进程的请求时，procd若是检查发现该init服务进程的配置/环境没有改变就可以不执行这个重启动作。

procd设计是为了替代以下功能的：

1. hotplug2：一个为嵌入式设备设计的动态设备管理子系统，hotplug2是一个Udev的极小替代实现。
2. busybox-klogd以及busybox-syslogd
3. busybox-watchdog

procd保持了对已存在的/etc/config格式的兼容性。

[这里](https://wiki.openwrt.org/doc/techref/procd#plugin_include__doc__techref__architecture)给出OpenWrt和桌面发行版以及安卓等系统上各种核心功能的实现模块。

#### dbus和ubus的区别

dbus太过臃肿，它的C API用起来很繁琐，并且要写大量样板代码。实际上，连dbus官方的API文档也对其C API有这样的说法：“如果你直接使用这些底层API，就是自找苦吃。”

ubus很轻量级并且从常规C代码中使用时很有优势，同样的其所有导出的API功能也在shell脚本中提供了，不需要再做额外工作就能用。

### 如何编写procd init脚本

procd的init脚本相对旧的init脚本（SysV）而言有以下不同：

1. procd预期服务在前端运行。
2. 使用了不同的shebang行。
3. 包含`USE_PROCD=1`明确表明使用procd。

比如：

```bash
#!/bin/sh /etc/rc.common

USE_PROCD=1
```

要启动一个任务，需要添加一个名为`start_service`的函数。`stop_srevice`只有在停止服务时要做特殊事情的情况下需要，否则procd会直接kill掉服务进程，`stop_srevice`函数在procd杀死服务进程后被调用。服务本身应该在前端运行。

```bash
start_service() {
  procd_open_instance
  procd_set_param command /sbin/your_service_daemon -b -a --foo

  # respawn automatically if something died, be careful if you have an alternative process supervisor
  # if process dies sooner than respawn_threshold, it is considered crashed and after 5 retries the service is stopped
  procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}

  procd_set_param env SOME_VARIABLE=funtimes  # pass environment variables to your process
  procd_set_param limits core="unlimited"  # If you need to set ulimit for your process
  procd_set_param file /var/etc/your_service.conf # /etc/init.d/your_service reload will restart the daemon if these files have changed
  procd_set_param netdev dev # likewise, except if dev's ifindex changes.
  procd_set_param data name=value ... # likewise, except if this data changes.
  procd_set_param stdout 1 # forward stdout of the command to logd
  procd_set_param stderr 1 # same for stderr
  procd_set_param user nobody # run service as user nobody
  procd_close_instance
}
```

#### 在配置文件或网络接口改变时触发procd

在以前的OpenWrt中，有一个叫ucitrack的系统调用用于跟踪配置文件，并且以来这些配置文件的进程会在需要时重启。这个机制也被ubus/procd替代了，并且扩展为当网络接口改变时允许通知服务（不一定重启）。这对dnsmasq，以及代理或路由等软件很有用，这些软件关心哪个网络接口是要使用的。

首先，简单的让你的服务依赖一个配置文件，可以在init脚本中添加一个`service_triggers()`函数：

```bash
service_triggers()
{
        procd_add_reload_trigger "uci-file-name"
}
```

这将会设置一个钩子：当文件/etc/config/uci-file-name的MD5值发生改变时执行`reload_config`会调用`/etc/init.d/<yourinitscript> reload`。你可以添加任意多你需要的配置文件，然后执行`reload_config`，procd将会关注所有这些文件的重载。注意：配置文件内容没有变化时不会执行重载动作。如果你想强制重载，还是要手动执行`/etc/init.d/<yourinitscript> reload`。

默认的，只有在最终配置的md5改变时，才会触发“reload”命令导致一次stop/start调用。如果你的程序没有用到参数，但是有一个生成的配置文件，或者直接分析一个uci配置文件，你应该通过这些方式添加到procd中：`procd_set_param file /var/etc/your_service.conf`，或者`procd_set_param file /etc/config/yourapp`。如果你没有任何文件或者参数可用，但是在reload被调用时依然需要强制服务进程重启，你可以编辑`reload_service`钩子如下：

```bash
reload_service()
{
        echo "Explicitly restarting service, are you sure you need this?"
        stop
        start
}
```

如果你希望你的服务依赖于网络的变化，简单的修改你的`service_triggers`部分：

```bash
service_triggers()
{
        procd_add_reload_trigger "uci-file-name" "second-uci-file"
        procd_add_network_trigger "lan"|"etho0" #这个目前还在实现中...
}
```

目前igmp和dnsmasq应该都用到这种方式。

#### 这些脚本如何工作的？

所有的参数都会包进json格式并通过ubus发给procd。

#### 调试

在脚本中添加`PROCD_DEBUG=1`，当start或者stop脚本是就可以看到调试信息。
