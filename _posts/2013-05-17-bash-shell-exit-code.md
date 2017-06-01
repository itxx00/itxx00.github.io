---
layout: post
title: "正确使用shell返回值"
description: "about shell exit code"
categories: [shell]
tags: [bash, exitcode]
---

> 编写shell脚本的时候，正确使用返回值是运维人员的基本操守

* Kramdown table of contents
{:toc .toc}

1.下表列出了常见shell命令的退出返回值：

|---
| 返回值 | 含义 | 示例 | 说明
|-|:-|:-:|-:
| 1  |  各种常见错误 |   let "var1 = 1/0"  |  shell里面最常见的错误返回值
| 2  |  shell内建功能使用错误 |   empty_function() {}  |  常见于关键字或者命令出错
| 126 |   命令无法执行 |   /dev/null  |  由于权限等导致的命令无法执行
| 127 |   命令无法找到 |   illegal_command |   一般是PATH环境变量不对等
| 128 |   退出返回值错误 |   exit 3.14159  |  返回值只能是整数，小数就不对了
| 128+n |    信号 "n"+128 |   kill -9 $PPID of script |   $? 即返回 137 (128 + 9)
| 130 |   ctrl+c 退出 |   Ctl-C  |  其实ctrl+c返回的是2 (130 = 128 + 2)
| 255* |   返回值超出可接受的范围 |   exit  -1 |   只能是 0 - 255

2. 下表列出了关于/etc/init.d/目录下启动控制脚本的标准返回值：

* 0    程序在运行或者服务状态OK
* 1    程序已经死掉，但是 pid文件仍在 /var/run目录下存在
* 2    程序已经死掉，但是lock文件仍在 /var/lock 目录下存在
* 3    程序没有运行
* 4    程序运行状态未知
* 5-99    供LSB扩展的保留段
* 100-149    供特定系统发行版使用的保留段
* 150-199    供特定程序使用的保留段
* 200-254    保留段

在写shell脚本的时候需要注意自定义的退出返回值最好不要与上面表格中所定义的重复，对于管理人员来说养成良好的习惯有助于遇到错误时作出正确的判断。
根据上表至少可以得出，在自定义返回值的时候：
- 最好不要用的：0-4 126-130 255
- 应避免使用的：5-99
- 可随意使用的：100-125 131-254

参考文档:
- http://tldp.org/LDP/abs/html/exitcodes.html
- http://refspecs.linux-foundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/iniscrptact.html
