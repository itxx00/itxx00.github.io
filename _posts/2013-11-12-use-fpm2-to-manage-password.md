---
layout: post
title: "使用fpm2来管理ssh密码"
description: ""
categories: [shell, system]
tags: [fpm2]
---

> 不知道有没有童鞋需要管理一堆ssh口令，5个以内靠脑子记可能是个好办法，但是如果10个100个的时候恐怕脑子记有点不太够用。


* Kramdown table of contents
{:toc .toc}

不知道有没有童鞋需要管理一堆ssh口令，5个以内靠脑子记可能是个好办法，但是如果10个100个的时候恐怕脑子记有点不太够用，
这个时候就需要借助外部工具来进行管理，当然你可以自己写个简单的脚本，把ssh账号密码写入一个list里面，
人懒且为了省事也可以使用一些现成的工具，下面就推荐一个图形界面的小工具给需要的童鞋：fpm2
fpm2全名Figaro's Paaword Manager 2，是一个开源软件，使用GNU General Public License Version 2 协议，这里是[官方地址](http://als.regnet.cz/fpm2/)，它还有android版本的。

在fedora里面可以直接yum安装：

`yum install fpm2`

安装完成后第一次运行需要你输入一个密码，今后每次启动fpm2的时候就用这个密码，默认如果密码输入错误次数超过3次，则你懂的。

打开fpm2后，一看就明白如何使用，它支持ssh/web以及自定义的密码管理，可以对管理的服务器进行分类，十分方便。其亮点是你可以根据自己的需要设置launcher，默认双击建立好的口令就会自动执行launcher定义的命令；

launcher里面将保存的账号密码和IP/URL定义为参数，`$a ip/url`  ，`$u  username`  ， `$p  password`， 有了这些变量后自定义launcher就很方便了。

但是其默认的ssh的launcher是不支持直接双击list里面的项目就登陆进服务器的，这个时候需要另外一个小工具sshpass，它的用处是登陆ssh的时候可以把密码作为参数传递给ssh客户端，而不需要交互式输入密码，这个工具代码托管在sf，fedora里也可以yum安装：

`yum install sshpass`

光有这两个工具还不够，下面是将两个工具完美结合起来的关键：

在fpm2的settings选项卡下有launcher设置项，打开之后你会发现默认的ssh launcher，下面是我自定义的ssh的加载器命令：

配置launcher

`gnome-terminal -e 'sh -c "'"sshpass -p '"'$p'"' ssh  -p 22 $u@$a;sudo -s"'"'`

特别注意里面的单双引号的写法，不然如果你的密码里有像%&$之类的特殊符号时是会出问题的，

`gnome-terminal -e 'xxxxx'` 这里是单引号

`sh -c "'" xxxxx "'"` 这里是两对双引号中包含的单引号

`sshpass -p '"' xxx '"'` 这里是两对单引号包含的双引号

ssh命令后面加个sudo -s的作用是当退出ssh连接时不会立即关闭当前的terminal终端

使用这个launcher可以通杀所有特殊字符的密码，在运行fpm2+sshpass的组合前请自己使用当前运行fpm2的账户登陆ssh一下远程服务器将服务器的publickey取回来，没有key的情况下sshpass无法工作的。至少我遇到的是这样。
