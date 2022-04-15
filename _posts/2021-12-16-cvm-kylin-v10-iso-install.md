---
layout: post
title: "CVM使用ISO镜像安装银河麒麟v10 arm系统"
categories: [system,os]
tags: [kylin, cvm]
---


* Kramdown table of contents
{:toc .toc}

> 背景：云上没有kylin的arm镜像,需要自己做一个

## 1 准备
iso: Kylin-Server-10-SP2-aarch64-Release-Build09-20210524.iso

一台arm的cvm, 一块数据盘

scp  Kylin-Server-10-SP2-aarch64-Release-Build09-20210524.iso  x.x.x.x:/kylin.iso

## 2 配置grub

修改grub配置增加从iso引导的入口，重启机器时从iso引导进入安装流程

```

# cat /etc/grub.d/40_custom
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.

menuentry 'Install Kylin Linux Advanced Server V10' --class red --class gnu-linux --class gnu --class os {
    set isolabel="Kylin-Server-10"
    set isofile="/kylin.iso"
    insmod iso9660
    loopback loop $isofile
    linux (loop)/images/pxeboot/vmlinuz inst.stage2=hd:LABEL=Kylin-Server-10 ro iso-scan/filename=$isofile console=tty0 video=efifb:off video=VGA-1:640x480-32@60me
    initrd (loop)/images/pxeboot/initrd.img
}

```

上面的参数从哪获取来？
1 
```
mount /kylin.iso /mnt
find /mnt -name grub.cfg
```
找到的内容作为linux行的参考

2
```
blkid /kylin.iso
```
可以获得isolabel信息

下一步
```
vi /etc/default/grub
#修改GRUB_TIMEOUT=60 增加timeout方便web vnc登录操作
grub2-mkconfig --ouput=/boot/grub2/grub.cfg
sync
reboot
```

## 3 开始装系统

系统会安装到数据盘，因为系统盘被iso占用，mount状态无法使用，必须有独立的数据盘用来装系统
注意安装cloud-init包。

## 4 制作云镜像

重启回到原先的系统
```
yum -y install qemu-img
qemu-img convert -f raw -O qcow2 /dev/vdb /kylin.qcow2
```
