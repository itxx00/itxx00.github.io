---
layout: post
title: "如何让ThinkPad自定义风扇转速"
description: ""
categories: [system]
tags: [thinkpad, fan]
---

> 笔记本长期处于高强度高负荷工作状态，手放到出风口的时候差点被烫伤，再不调整下恐怕到了夏天是彻底扛不住了。于是网上搜了下资料，还好thinkpad一直都有黑客在用...

* Kramdown table of contents
{:toc .toc}

还好thinkpad一直都有黑客在用，因此也不缺乏内核级别的支持，下面记录下配置方法，怕今后再要捣鼓的时候忘记。首先thinkpad有一个专用的acpi驱动叫thinkpad_acpi的内核模块，这个在centos里面已经自带了，它的项目地址http://ibm-acpi.sf.net/。上面有列出支持哪些哪些型号哪些功能。

你可以通过lsmod命令查看是否已经加载了此模块:

`lsmod|grep think`

这个模块加载之后可以通过proc内的文件来查看风扇的运行状态：

`cat /proc/acpi/ibm/fan`

如果有，那么进入下一步，添加模块的加载选项，创建模块配置文件：

```
[root@server ~]# vi /etc/modprobe.d/thinkpad_acpi.conf
[root@server ~]# cat /etc/modprobe.d/thinkpad_acpi.conf
options thinkpad_acpi experimental=1 fan_control=1
```

上面的配置将打开自定义风扇转速的开关。

这个时候需要重新加载模块：

```
modprobe -r thinkpad_acpi && modprobe thinkpad_acpi
```

然后再查看下proc里面fan文件的状态：

```
cat /proc/acpi/ibm/fan
```

此时你会发现，信息有变化。

```
[root@server ~]# cat /proc/acpi/ibm/fan
status: enabled
speed: 5485
level: auto
commands: level <level> (<level> is 0-7, auto, disengaged, full-speed)
commands: enable, disable
commands: watchdog <timeout> (<timeout> is 0 (off), 1-120 (seconds))
```

然后我们就可以通过command来控制风扇的转速啦，auto表示自动，disengaged和full-speed一个效果，也可以设置0-7的等级，0表示停止，感兴趣可以试试，反正我不敢试，

```
echo "level  full-speed"  > /proc/acpi/ibm/fan
```
注意level和full-speed之间只有一个空格..

再查看fan文件的状态，或者听听风扇转动的声音，你就会发现有明显的变化了，如果模块没有添加fan_control=1的参数的话往/proc/acpi/ibm/fan里面echo信息是不会成功的，会报如下错误：

```
[root@server ~]# echo "level 5" > /proc/acpi/ibm/fan
bash: echo: write error: Invalid argument
```

总结：风扇使劲转起来了，再也不用担心手被烫伤了！
