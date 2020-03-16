---
layout: post
title: "CentOS8安装后grub菜单增加windows入口"
description: "默认安装完不会自动识别其他系统，需要手工添加"
categories: [os]
tags: [os,centos,grub,centos8 ]
---

> 电脑双系统centos+windows，安装完centos8之后默认没有引导windows的入口，按照下面方法手搓即可。

* Kramdown table of contents
{:toc .toc}
## 1 启动进入centos
查看磁盘分区信息，如下：
```fdisk -l```

```
# fdisk -l
Disk /dev/sda: 238.5 GiB, 256060514304 bytes, 500118192 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x297f5cef

Device     Boot     Start       End   Sectors   Size Id Type
/dev/sda1  *         2048 250058751 250056704 119.2G  7 HPFS/NTFS/exFAT
/dev/sda2       250058752 393418751 143360000  68.4G  7 HPFS/NTFS/exFAT
/dev/sda3       393418752 394442751   1024000   500M 83 Linux
/dev/sda4       394442752 500117503 105674752  50.4G  5 Extended
/dev/sda5       394444800 500117503 105672704  50.4G 83 Linux
 ```
通过fdisk结果看到windows第一个partion在sda1，对应grub的磁盘索引编号是hd0,1,接下来编辑grub配置文件，自定义配置路径：
```
 vi  /etc/grub.d/40_custom
```
配置示例如下：
```
 #!/bin/sh
 exec tail -n +3 $0
 # This file provides an easy way to add custom menu entries.  Simply type the
 # menu entries you want to add after this comment.  Be careful not to change
 # the 'exec tail' line above.

 menuentry "Windows" {
         set root=(hd0,1)
         chainloader +1
         }
```

保存并执行以下命令使自定义配置生效：
```
grub2-mkconfig --output=/boot/grub2/grub.cfg
```
OVER.
