---
layout: post
title:  "在openSUSE中安装redmine"
date:	20 Nov 2014 08:58:21 +0800
categories: redmine
tags: redmine openSUSE
---

*   从[官网][web]下载最新的redmine压缩包。
*   参考[官方安装文档][install]，按照需求安装所需的其他软件包以及redmine。

[web]: http://www.redmine.org "redmine官网"
[install]: http://www.redmine.org/projects/redmine/wiki/RedmineInstall "redmine安装指南"

## 安装必须的基础软件及包

我的操作系统是openSUSE13.1 64位版本，以下内容都是基于此系统操作，部分操作命令需要root权限。

1.  redmine基于*ROR* (Ruby on Rails)，所以需要ROR环境，系统中已经安装了*ruby* 2.0.0（ruby包中就已经包含了*gem*）。已经安装了ruby的情况下只需要通过gem安装rails即可。执行`gem install rails`即可安装好rails。

    > 国内访问gem官方仓库速度太慢，可以使用taobao的gem仓库镜像`gem sources -r https://rubygems.org/; gem sources -a http://ruby.taobao.org`。

2.  安装数据库软件。redmine支持常见的多个数据库，由于我的系统中默认已经安装了*mariadb*，该软件是mysql的分支且几乎完全兼容mysql，所以以下操作和使用mysql相同。

    默认的mysql或者mariadb中数据库管理员账户root是没有密码的，为安全考虑，先给root账户设置密码`mysqladmin -u root password <password for root>`，如要修改root密码`mysqladmin -u root password <oldpassword> <newpassword>`.

    按照建议安装mysql的C binding，`gem install mysql2`.

## 安装redmine及其依赖

1.  参考[官方安装文档][install]，创建数据库以及用户并设置用户对数据库的权限。下面的命令是针对mysql的操作，注意将数据库名，用户名以及密码替换成你自己的。

        CREATE DATABASE redmine CHARACTER SET utf8;
        CREATE USER 'redmine'@'localhost' IDENTIFIED BY 'my_password';
        GRANT ALL PRIVILEGES ON redmine.* TO 'redmine'@'localhost';

2.  解压从官网下载的redmine（一般解压到/srv/www目录下，这个可以随自己的需求解压到哪里都可以）。

    根据上一步创建的数据库配置redmine，进入redmine根目录，复制*config/database.yml.example*为*config/database.yml*，并修改该配置文件如下。

    > production:  
    >   adapter: mysql  
    >   database: redmine  
    >   host: localhost  
    >   username: redmine  
    >   password: my_password

3.  安装依赖，redmine通过Bundler管理gem依赖，所以要先安装bundler。

    安装Bundler，执行`gem install bundler`。

    进入解压后的redmine根目录，执行`bundle install --without development test`，可以先修改Gemfile中的source以加快速度。这一步的安装中会编译一些包，如果提示错误，一般是系统中缺少编译需要的库或者头文件，根据提示信息给系统安装相关的devel包（使用zypper install）即可。

4.  生成事务存储安全凭证。

    进入redmine根目录，执行`rake generate_secret_token`。

5.  创建数据库图表对象。

    进入redmine根目录，执行`RAILS_ENV=production rake db:migrate`

6.  填充数据库默认数据。

    进入redmine根目录，执行`RAILS_ENV=production rake redmine:load_default_data`。

7.  最后，确保你运行redmine的用户对redmine应用的文件都相应的读写权限。

## 启动redmine

在redmine官方文档页面有一些文章介绍不同的系统上redmine安装教程，其中openSUSE系统可以参见这里[openSUSE上如何安装redmine][openSUSE_redmine]。参考文中的启动脚本启动redmine，不过该示例脚本是用ruby自带的一个web服务器启动，该工具用于测试及小范围环境使用，如果要在企业应用中启动redmine，最好使用apacha或nginx等专业的web服务器驱动redmine，相应的操作也比较复杂，不过网上有此类教程可以自行参考。

[openSUSE_redmine]: http://www.redmine.org/projects/redmine/wiki/HowTo_Install_Redmine_on_openSUSE "openSUSE上安装redmine"

## 备份及升级redmine

参考官方文档[redmine upgrade][redmine_upgrade]即可，此处不再赘述。

[redmine_upgrade]: http://www.redmine.org/projects/redmine/wiki/RedmineUpgrade "redmine备份及升级"
