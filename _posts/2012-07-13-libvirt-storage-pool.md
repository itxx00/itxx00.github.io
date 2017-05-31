---
layout: post
title: "libvirt中storage pool的管理"
description: "build storage pool with libvirt"
categories: [virt]
tags: [libvirt, storage pool]
redirect_from:
  - /2012/07/13/
---

> 简单演示如何在libvirt中创建storage pool

* Kramdown table of contents
{:toc .toc}

 
libvirt提供了存储管理的功能，可以管理的存储类型有 目录  lvm逻辑卷 磁盘 iscsi存储  scsi存储  mpath netfs等，这里以最基本的目录类型为例

基本概念：

在libvirt里保存虚拟机磁盘镜像的目录或设备称作存储池  即pool  ，每个虚拟机所使用的虚拟磁盘镜像称作卷 即vol ，vol是存储在pool里面的。

我们可以使用命令行的virsh工具来管理，pool有两种基本状态：活动和非活动，查看当前存储池的状态

```
virsh # pool-list --all 
Name State Autostart 
-----------------------------------------
default inactive yes 
disk active yes
```

新建一个基于目录的存储池bigpool 存储路径为 /bigpool 

```
virsh # pool-define-as bigpool dir - - - - /bigpool
Pool bigpool defined
```
这时候pool仅仅是定义出来了，可以用pool-list --all查看到。但是相应的目录是不存在的，接着需要建立这个pool

```
virsh # pool-build bigpool
Pool bigpool built
```

这个时候才是真正的建立起这个pool，libvirt会自动创建/bigpool目录，并设置相应的权限，如果你有用selinux作为libvirt的安全措施的话它还能自动设置上下文

```
# ls -Zld /bigpool/
drwx------ 2 ? root root 4096 Jul 13 12:54 /bigpool/
```

我这里由于没有使用selinux所以没有上下文的

 pool创建好之后就可以启动了

```
virsh # pool-start bigpool
Pool bigpool started

virsh # pool-list --all 
Name State Autostart 
-----------------------------------------
bigpool active no 
default inactive yes 
disk active yes
```

还可以设置pool为自动启动

```
virsh # pool-autostart bigpool
Pool bigpool marked as autostarted

virsh # pool-list --all 
Name State Autostart 
-----------------------------------------
bigpool active yes 
default inactive yes 
disk active yes
```

基于目录的一个存储池就这样建立完成了，是不是很简单？

 
