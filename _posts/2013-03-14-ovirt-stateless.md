---
layout: post
title: "Ovirt中的stateless实现机制"
description: "简单分析ovirt的stateless实现机制"
categories: [virt, system]
tags: [ovirt, stateless]
---

> 我发现ovirt的node也就是运行虚拟机的主机被设计成了这样：整个根文件系统是只读的，只有部分配置文件被独立出来放到了另外的分区，问了几位IBM和红帽的工程师，才知道这叫stateless，也就是是无状态模式

* Kramdown table of contents
{:toc .toc}

## Ovirt简介
这是来自centos wiki中的描述：
> oVirt 是个管理虚拟化的应用程序。言下之意就是你可利用 oVirt 的管理界面（oVirt 引擎）来管理硬件节点、存储及网络资源，并部署及监控在你的数据中心内运行的虚拟机器。
> 如果你熟识 VMware 的产器，oVirt 在理念上与 vSphere 类同。Red Hat 企业级虚拟化产品以 oVirt 作为基础，这个上游计划内开发的新功能，日后亦会在获支持的产品内出现。

ovirt是一套虚拟化管理的系统，其对标产品是vmware的vsphere，它分为ovirt-engine和ovirt-node，可以用ovirt的这套系统来管理虚拟机群，现在虚拟化相关产品众多，微软，vmware，oracle等都在做这方面的产品，于是IBM红帽ubuntu等厂商就联合起来开始搞这个东西了，红帽很早就开始投入kvm了，红帽做的是RHEV的整套系统，也分为node和engine只是名字不一样，后来干脆就把node的RHEV-H给贡献出来，大家一起搞ovirt了。前几天我也试用了一下ovirt和RHEV，发现他们的node也就是运行虚拟机的主机被设计成了这样: 整个根文件系统是只读的，只有部分配置文件被独立出来放到了另外的分区，问了几位IBM和红帽的工程师，才知道这叫stateless，无状态，这么做的好处是运行环境和存储分离，提高整体可用性。在分析了ovirt中的stateless实现机制之后，下面将在centos6上尝试手工配置，过程中请教了几位ovirt的开发,再次表示感谢

##  关于stateless
这种stateless的设计来自很早之前红帽在fedora里面做的尝试，目的是把系统做成liveCD，下面是一些关于stateless的描述：

read-only root file system(stateless linux)
Readonly root support.
This was add to Fedora for Stateless Linux, i.e. for creating live Fedora CDs.
How to use:
   * Edit `/etc/sysconfig/readonly-root`. Set 'READONLY' to 'yes'.
   * Add any exceptions that need to be writable that aren't in the stock `/etc/rwtab` to an /etc/rwtab.d file. (See below)
How it works:
   * On boot, we mount a tmpfs (by default, at /var/lib/stateless/writable), and then parse `/etc/rwtab` and `/etc/rwtab.d/*` for things to put there.
These files have the format:
`<type>  <path>`

* Types are as follows:
  * empty: An empty path. Example:
              `empty     /tmp`
  * dirs: A directory tree that is copied, empty. Example:
              `dirs      /var/run`
  * files: A file or directory tree that is copied intact. Example:
              `files     /etc/resolv.conf`

A stock rwtab is shipped with common things that need mounted.
When your computer comes back up, the root and any other system
partitions will be mounted read-only. All the files and directories
listed in `/etc/rwtab` will be mounted read-write on a tmpfs filesystem.
You can add additional files and directories to rwtab to make them
writable after reboot.

Note that this system is stateless. When you reboot again, everything
written to the tmpfs filesystem vanishes and the system will be exactly
as it was the last time it was booted. You could add a writable
filesystem on disk or NFS for writing files you want to retain after
rebooting.

Take a look at `/etc/rc.d/rc.sysinit` to see how the magic is done.
This capability is a "technology preview" (beta) and is buggy. Note that
`/etc/mtab` and thus "mount" do not show the complete list of filesystems
because the /etc directory is on a read-only filesystem. /proc/mounts
always shows the correct mount information. You could update `/etc/mtab`
from /proc/mounts to correct it both after boot and after running the
mount or umount commands to change mounts.

Run `fgrep -v rootfs /proc/mounts >/etc/mtab` to correct `/etc/mtab`.
Note that mounting or symlinking /proc/mounts to /etc/mtab causes other
problems such as breaking the df command.

You can change your read-only root filesystem to read-write mode
immediately with this command run by the root user:
`mount -n -o remount,rw /`

## 分析其实现机制

下面把分析过程记录下来：

首先我看到的是fstab文件

/etc/fstab
~~~
/dev/root / ext2 defaults,ro,noatime 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
tmpfs /dev/shm tmpfs defaults 0 0
proc /proc proc defaults 0 0
sysfs /sys sysfs defaults 0 0
/dev/HostVG/Config /config ext4 defaults,noauto,noatime 0 0
debugfs /sys/kernel/debug debugfs 0 0
/dev/HostVG/Swap swap swap defaults 0 0
/dev/HostVG/Logging /var/log ext4 defaults,noatime 0 0
/dev/HostVG/Data /data ext4 defaults,noatime 0 0
/data/images /var/lib/libvirt/images bind bind 0 0
/data/core /var/log/core bind bind 0 0
~~~

这里要注意的地方就是root后面的ro，也就是说被挂载成了只读，这说明stateless是在挂载磁盘之前就已经完成，那肯定跟系统启动相关，
接着我们找到rc.sysinit这个在系统启动阶段执行的脚本中下面这段内容:

/etc/rc.d/rc.sysinit
~~~bash
for file in /etc/statetab /etc/statetab.d/* ; do
    is_ignored_file "$file" && continue
    [ ! -f "$file" ] && continue

    if [ -f "$STATE_MOUNT/$file" ] ; then
        mount -n --bind "$STATE_MOUNT/$file" "$file"
    fi

    for path in $(grep -v "^#" "$file" 2>/dev/null); do
        mount_state "$path"
        [ -n "$SELINUX_STATE" -a -e "$path" ] && restorecon -R "$path"
    done
done
~~~

这段shell里面牵扯到了个/etc/statetab,而且通过对比普通系统还发现其他地方的不同


diff rc.sysinit /etc/rc.sysinit
~~~
102a103,108
> elif [[ "$system_release" =~ "CentOS" ]]; then
> [ "$BOOTUP" = "color" ] && echo -en "\\033[0;36m"
> echo -en "CentOS"
> [ "$BOOTUP" = "color" ] && echo -en "\\033[0;39m"
> PRODUCT=$(sed "s/CentOS \(.*\) \?release.*/\1/" /etc/system-release)
> echo " $PRODUCT"
499c505
< action $"Mounting local filesystems: " mount -a -t nonfs,nfs4,smbfs,ncpfs,cifs,gfs,gfs2,noproc,nosysfs,nodevpts -O no_netdev
---
> action $"Mounting local filesystems: " mount -a -t nonfs,nfs4,smbfs,ncpfs,cifs,gfs,gfs2 -O no_netdev
501c507
< action $"Mounting local filesystems: " mount -a -n -t nonfs,nfs4,smbfs,ncpfs,cifs,gfs,gfs2,noproc,nosysfs,nodevpts -O no_netdev
---
> action $"Mounting local filesystems: " mount -a -n -t nonfs,nfs4,smbfs,ncpfs,cifs,gfs,gfs2 -O no_netdev
~~~

接着看看这个/etc/statetab文件是个什么样子的：

/etc/statetab
~~~
# A list of paths which should be bind-mounted from a
# partition dedicated to persistent data
#
# See $STATE_LABEL in /etc/sysconfig/readonly-root
#
# Examples:
#
# /root
# /etc/ssh
# /var/spool/mail
~~~

顺藤摸瓜我们发现和/etc/sysconfig/readonly-root这个文件有关，从文件名上即可得知和只读的关联：

/etc/sysconfig/readonly-root
~~~
Set to 'yes' to mount the system filesystems read-only.
READONLY=yes
# Set to 'yes' to mount various temporary state as either tmpfs
# or on the block device labelled RW_LABEL. Implied by READONLY
TEMPORARY_STATE=no
# Place to put a tmpfs for temporary scratch writable space
RW_MOUNT=/var/lib/stateless/writable
# Label on local filesystem which can be used for temporary scratch space
RW_LABEL=stateless-rw
# Options to use for temporary mount
RW_OPTIONS=
# Label for partition with persistent data
STATE_LABEL=CONFIG 
# Where to mount to the persistent data
STATE_MOUNT=/config
# Options to use for peristent mount
STATE_OPTIONS=
# NFS server to use for persistent data?
CLIENTSTATE=
~~~

就是这个了，首先它把READONLY设置为yes，然后使用设备的LABEL号来指定需要挂载为读写的设备，然后就是挂载的位置STATE_MOUNT

同时还有/etc/rwtab以及rwtab.d目录与这个有关：
/etc/rwtab
~~~
#files /etc/adjtime
#files /etc/ntp.conf
#files /etc/resolv.conf
#files /etc/lvm/.cache
#files /etc/lvm/archive
#files /etc/lvm/backup
~~~

上面这几项在ovirt-node里是被注释掉了的，它使用自己的方式来变更这几个文件

/etc/rwtab.d/ovirt
~~~
files /etc
dirs /var/lib/multipath
dirs /var/lib/net-snmp
dirs /var/lib/dnsmasq
files /root/.ssh
dirs /root/.uml
dirs /root/.virt-manager
dirs /home/admin/.virt-manager
files /var/cache/libvirt
files /var/empty/sshd/etc/localtime
files /var/lib/libvirt
files /var/lib/multipath
files /var/cache/multipathd
empty /mnt
empty /live
files /boot
empty /boot-kdump
empty /cgroup
~~~

上面这些是ovirt自定义的。最后就是看/config下面的files文件:

/config/files
~~~
/etc/fstab
/etc/shadow
/etc/default/ovirt
/etc/ssh/ssh_host_key
/etc/ssh/ssh_host_key.pub
/etc/ssh/ssh_host_dsa_key
/etc/ssh/ssh_host_dsa_key.pub
/etc/ssh/ssh_host_rsa_key
/etc/ssh/ssh_host_rsa_key.pub
/etc/iscsi/initiatorname.iscsi
/etc/sysconfig/network-scripts/ifcfg-eth0
/etc/sysconfig/network-scripts/ifcfg-breth0
/etc/sysconfig/network-scripts/ifcfg-eth1
/etc/ntp.conf
/etc/sysconfig/network
/etc/hosts
/etc/shadow
/etc/ssh/sshd_config
~~~

此列表内的文件就是rc.sysinit需要读取的，把他们一个个挂载为读写，这样就实现了可修改配置文件白名单，于是我就动手修改了系统里的这些文件，重启，一切看似正常，但是当关机的时候新的问题出现了，关机的时候提示/etc无法被卸载，为什么呢，然后就找到与关机有关的脚本：

diff /etc/rc.d/init.d/halt.orig /etc/rc.d/init.d/halt
~~~
141c141
< LANG=C __umount_loop '$2 ~ /^\/$|^\/proc|^\/dev/{next}
---
> LANG=C __umount_loop '$2 ~ /^\/$|^\/proc|^\/etc|^\/dev/{next}
~~~

原来在halt文件里也做了“手脚”，把/etc给加了进去，重启，修改，关机，一切正常。

## 参考文档
下面是可以参考的文档

- http://fedoraproject.org/wiki/StatelessLinux/PrepareImage
- http://fedoraproject.org/wiki/StatelessLinux/HOWTO
- http://blog.csdn.net/jcwkyl/article/details/6120547
- http://plone.lucidsolutions.co.nz/linux/io/using-centos-5.2-stateless-linux-support-on-a-flash-based-root-filesystem
- FYI: http://ovirt.org/wiki/File:Ovirt-node.pdf (page 26)

