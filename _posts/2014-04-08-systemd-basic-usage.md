---
layout: post
title: "Systemd基本使用介绍"
description: ""
categories: [system]
tags: [systemd]
---

> 但是由于计算机软硬件的不断发展，人们逐渐发现sysvinit所提供的功能已经无法满足当前的需求，服务多年的sysvinit终将过时，于是一些替代的方案开始出现，这其中包括upstart ，systemd等。

* Kramdown table of contents
{:toc .toc}

## 关于PID 1
在linux的世界里，第一个启动到用户空间的进程名叫init，其pid为1，init进程启动完毕后，它相当于其他进程的根，负责将其他的服务进程启动起来，最终启动成为一个完整可用的系统。而提供这个init程序的软件名为sysvinit，它在整个linux系统中所担任的角色重要程度无可厚非。但是由于计算机软硬件的不断发展，人们逐渐发现sysvinit所提供的功能已经无法满足当前的需求，服务多年的sysvinit终将过时，于是一些替代的方案开始出现，这其中包括upstart ，systemd等。

## init的责任
为什么说sysvinit会过时呢？我们从用户需求的角度来看不难发现，其实我们对init的最原始最核心需求是，将系统从内核引导至用户空间，而由于现在硬件水平的发展，使得我们在单纯的原始需求之上产生了新的或者更多的需求，那就是不仅要引导系统，而且要快速的引导系统。而早在sysvinit诞生的那个时候启动速度的重要程度似乎不是很高，所以它在这方面显得有些老是很自然的。现在让我们继续思考一下如何才能实现快速引导。试想，当一个系统启动的时候需要启动长长的一列服务，这样的系统启动速度应该不会快到哪里去，如大多数人使用桌面系统的情况一样，一个加快系统启动速度的最直接方法就是减少启动项，把不必要的启动项禁止掉。而另外一个方法则实现起来不像第一个这么简单，那就是并行启动。并行启动可以带来的速度提升显而易见，我们只需要尽可能保证让更多的服务可以并行启动即可。而从其他方面来看sysvinit由于对脚本的依赖导致启动完毕所有服务过程中需要大量的执行外部命令，这一点也将导致引导速度的变慢。

## 后起之秀
当一项标准已经无法满足当下发展的需求时，最好的办法是打破陈规让自己变成新的标准。systemd就是这么做的，因为现实的需要它已经不能顾及POSIX标准了，以不至于阻碍其更好的发展，而systemd在设计之初也和苹果的launchd在某些地方不谋而合。尽管在最初的时候这种做法没有得到所有人的认可甚至是惹恼了一些家伙，但是现在看来systemd已经成为了事实上的标准了，现在已经采用systemd的发行版有fedora，opensuse，debian，ubuntu，rhel7，archlinux等，尽管在systemd的推广过程中发生了很多曲折，但最终大家的选择是一致的---朝着更好的方向发展而不是墨守陈规。
在功能上，systemd不仅仅是解决了现有程序的种种问题，而且包含了大量新的特性，这意味着系统中原本需要的很多组件现在也都可以下岗了，下面来看看systemd的一些特性：

* 服务管理 - 提供了统一的启动/停止/重启/重载 而不再需要编写一大坨脚本来干这种原本就应该十分简单基本且重要的事情，取而代之的是更为简洁的配置文件，这也表示今后的系统中各种service启动控制的统一性，使得daemon服务开发者免于编写各种看上去参差不齐的启动脚本，增加了通用性。
* socket管理 - 支持监听socket，这使得socket的监听和服务本身可以相互独立，一方面可以以此提高服务启动速度，另外一方面还可以节约系统开销。
* 设备管理 - 可以配合udev规则对设备进行管理，还可以配合/etc/fstab文件实现磁盘的挂载管理，甚至实现更高级的自动挂载；
* target分组 - 把不同的unit组合起来，组合到同一个target里面，完全取代sysvinit的运行等级的概念；
* 状态快照 - 可以对当前系统中的unit状态进行快照，这个概念有些类似于系统的休眠与恢复，你可以让系统从一个状态切换进入另外一个状态；

## systemd管理入门
下面我们将通过一些例子来演示一些基本的管理命令，这些命令对于系统管理员来说至关重要。

### 1  如何检查启动项
任何启动项，只要是在系统启动时有被执行到，不论启动成功还是失败，systemd都能够记录下他们的状态，直接执行不带参数的systemctl命令即可观察到，如下：
~~~bash
# systemctl
UNIT                                             LOAD   ACTIVE SUB       DESCRIPTION
proc-sys-fs-binfmt_misc.automount                loaded active waiting   Arbitrary Executable File Formats File System Aut
sys-devices-pci...0-backlight-acpi_video1.device loaded active plugged   /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0
sys-devices-pci...0-backlight-acpi_video0.device loaded active plugged   /sys/devices/pci0000:00/0000:00:02.0/backlight/ac
sys-devices-pci...00-0000:00:19.0-net-em1.device loaded active plugged   82579LM Gigabit Network Connection
sys-devices-pci...d1.4:1.0-bluetooth-hci0.device loaded active plugged   /sys/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1
sys-devices-pci...000:00:1b.0-sound-card0.device loaded active plugged   6 Series/C200 Series Chipset Family High Definiti
sys-devices-pci...0000:03:00.0-net-wlp3s0.device loaded active plugged   RTL8188CE 802.11b/g/n WiFi Adapter
sys-devices-pci...-0:0:0:0-block-sda-sda1.device loaded active plugged   ST9500420AS
sys-devices-pci...-0:0:0:0-block-sda-sda2.device loaded active plugged   ST9500420AS
sys-devices-pci...-0:0:0:0-block-sda-sda3.device loaded active plugged   ST9500420AS
sys-devices-pci...-0:0:0:0-block-sda-sda5.device loaded active plugged   ST9500420AS
sys-devices-pci...-0:0:0:0-block-sda-sda6.device loaded active plugged   ST9500420AS
sys-devices-pci...-0:0:0:0-block-sda-sda7.device loaded active plugged   LVM PV EKHM59-PY9G-AoRX-Nr9k-nnxN-XxxO-DFcj4N on
sys-devices-pci...0:0:0-0:0:0:0-block-sda.device loaded active plugged   ST9500420AS
sys-devices-pci...-1:0:0:0-block-sdb-sdb1.device loaded active plugged   KINGSTON_SVP200S360G
sys-devices-pci...-1:0:0:0-block-sdb-sdb2.device loaded active plugged   LVM PV rlRqSb-IlQn-DJQi-i7fg-sUV5-3bjI-g2npg7 on
sys-devices-pci...1:0:0-1:0:0:0-block-sdb.device loaded active plugged   KINGSTON_SVP200S360G
~~~

要检查具体的服务，则使用status选项加上服务名即可，如下：
~~~bash
# systemctl status libvirtd
libvirtd.service - Virtualization daemon
   Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled)
   Active: active (running) since 一 2014-04-07 19:10:30 CST; 9min ago
     Docs: man:libvirtd(8)
           http://libvirt.org
 Main PID: 1673 (libvirtd)
   CGroup: /system.slice/libvirtd.service
           ├─1673 /usr/sbin/libvirtd
           └─1804 /sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf


4月 07 19:10:33 localhost.localdomain dnsmasq[1804]: using nameserver 218.6.200.139#53
4月 07 19:10:33 localhost.localdomain dnsmasq[1804]: using nameserver 61.139.2.69#53
4月 07 19:10:33 localhost.localdomain dnsmasq[1804]: using local addresses only for unqualified names
4月 07 19:10:33 localhost.localdomain dnsmasq[1804]: read /etc/hosts - 3 addresses
4月 07 19:10:33 localhost.localdomain dnsmasq[1804]: read /var/lib/libvirt/dnsmasq/default.addnhosts - 0 addresses
4月 07 19:10:33 localhost.localdomain dnsmasq-dhcp[1804]: read /var/lib/libvirt/dnsmasq/default.hostsfile
Hint: Some lines were ellipsized, use -l to show in full.
~~~

以上这些信息向我们展示了服务的运行时间，当前状态，相关进程pid，以及最近的有关该服务的日志信息，是不是很方便？

### 2 找出每项服务所对应的进程id
对于系统管理来说这是最常见的工作，但是在sysvinit时代我们只能借助例如ps命令等来完成，而systemd已经考虑到了系统管理员的需求，于是下面的命令诞生了：

~~~bash
# systemd-cgls
├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
├─user.slice
│ ├─user-1000.slice
│ │ ├─session-1.scope
│ │ │ ├─1806 gdm-session-worker [pam/gdm-password]
│ │ │ ├─1820 /usr/bin/gnome-keyring-daemon --daemonize --login
│ │ │ ├─1822 gnome-session
│ │ │ ├─1830 dbus-launch --sh-syntax --exit-with-session
│ │ │ ├─1831 /bin/dbus-daemon --fork --print-pid 4 --print-address 6 --session
│ │ │ ├─1858 /usr/libexec/at-spi-bus-launcher
│ │ │ ├─1862 /bin/dbus-daemon --config-file=/etc/at-spi2/accessibility.conf --nofork --print-address 3
│ │ │ ├─1865 /usr/libexec/at-spi2-registryd --use-gnome-session
│ │ │ ├─1872 /usr/libexec/gvfsd
... ...
~~~

现在我们可以很清晰的看到哪项服务启动了哪些进程，对于系统管理来说十分有用；

### 3 如何正确的杀死服务进程
在sysvinit的时代，如果需要结束一个服务及其启动的所有进程，可能会遇到一些糟糕的进程无法正确结束掉，即便是我们使用kill，killall等命令来结束它们，到了systemd的时代一切都变得不一样，systemd号称是第一个能正确的终结一项服务的程序，下面来看看具体如何操作的：

`systemctl kill crond.service`

或者指定一个信号发送出去

`systemctl kill -s SIGKILL crond.service`

例如可以像这样执行一次reload操作

`systemctl kill -s HUP --kill-who=main crond.service`

### 4 如何停止和禁用一项服务
下面我们回顾一下在sysvinit时代执行下面的命令所实现的功能
```
# service ntpd stop
# chkconfig ntpd off
```

很简单，首先停止服务，其次禁用服务
那么在systemd中应该如何操作呢？
```
# systemctl stop ntpd.service
# systemdctl disable ntpd.service
```
很显然systemctl命令已经取代了service 和chkconfig两个命令的位置，不光如此，我们还能将服务设置为连人工也无法启动：
```
# ln -s /dev/null  /etc/systemd/system/ntpd.service
# systemctl daemon-reload
```

### 5 检查系统启动消耗时间
在过去我们可以借助第三方工具来实现对系统启动过程的耗时跟踪，现在这个功能已经集成到systemd当中，可以十分方便的了解到系统启动时在各个阶段所花费的时间：
```bash
# systemd-analyze
Startup finished in 1.669s (kernel) + 1.514s (initrd) + 7.106s (userspace) = 10.290s
以上信息简要的列出了从内核到用户空间整个引导过程大致花费的时间，如果要查看具体每项服务所花费的时间则使用如下命令：
# systemd-analyze blame
          6.468s dnf-makecache.service
          5.556s network.service
          1.022s plymouth-start.service
           812ms plymouth-quit-wait.service
           542ms lvm2-pvscan@8:7.service
           451ms systemd-udev-settle.service
           306ms firewalld.service
           246ms dmraid-activation.service
           194ms lvm2-pvscan@8:18.service
           171ms lvm2-monitor.service
           145ms bluetooth.service
           135ms accounts-daemon.service
           113ms rtkit-daemon.service
           111ms ModemManager.service
           104ms avahi-daemon.service
           102ms systemd-logind.service
            79ms systemd-vconsole-setup.service
            77ms acpid.service
```

输出结果类似上面这些内容，至此我们可以看到系统里每一项服务的启动时间，据此可以对系统引导过程了如指掌，如果你觉得这还不够直观，那么可以将结果导出到图像文件里面：

`systemd-analyze plot >systemd.svg`

该命令会将系统的启动过程输出到一张svg图像上面，更直观的展现出各个服务启动所花费的时间，在我们需要分析和优化服务启动项的时候很有帮助。

### 6 查看各项服务的资源使用情况
和top命令不一样，top命令更侧重于展示以进程为单位的资源状态，而systemd提供了一个命令来方便的查看每项服务的实时资源消耗状态：
~~~bash
# systemd-cgtop
Path                                              Tasks   %CPU   Memory  Input/s Output/s
/                                                   199   12.3     1.9G        -        -
/system.slice/ModemManager.service                    1      -        -        -        -
/system.slice/abrt-oops.service                       1      -        -        -        -
/system.slice/abrt-xorg.service                       1      -        -        -        -
/system.slice/abrtd.service                           1      -        -        -        -
/system.slice/accounts-daemon.service                 1      -        -        -        -
/system.slice/acpid.service                           1      -        -        -        -
/system.slice/alsa-state.service                      1      -        -        -        -
/system.slice/atd.service                             1      -        -        -        -
/system.slice/auditd.service                          3      -        -        -        -
/system.slice/avahi-daemon.service                    2      -        -        -        -
/system.slice/bluetooth.service                       1      -        -        -        -
/system.slice/chronyd.service                         1      -        -        -        -
/system.slice/colord.service                          1      -        -        -        -
/system.slice/crond.service                           1      -        -        -        -
~~~

### 7 我们必须要注意到的配置文件变更
systemd考虑到各种发行版本的用户使用习惯，尽量提供更为通用的配置文件以方便各家统一，方便用户使用，下面列出一些基本的统一的配置文件：
* /etc/hostname，debian和redhat在这个配置文件上的差异导致了系统管理或多或少的不便，此文件的统一意义重大
* /etc/vconsole.conf，此文件用来统一管理console和键盘映射
* /etc/locale.conf 配置系统环境语系
* /etc/modules-load.d/*.conf 内核模块加载配置文件
* /etc/sysctl.d/*.conf 内核参数配置文件，对/etc/sysctl.conf的扩充
* /etc/tmpfiles.d/*.conf 运行态临时文件配置
* /etc/os-release  /etc/machine-id  /etc/machine-info 这三个文件的统一对系统管理员来说也是意义深远，它让我们有了统一的检测发行版本等信息的入口

以上内容仅仅让各位对systemd形成基本的认识，我们将在后期的文章中更加深入地讨论systemd的特性。最后，再次感谢作者Lennart的贡献。

> 参考链接：
> * http://cgit.freedesktop.org/systemd/
> * http://0pointer.de/blog/projects/
> * https://linuxtoy.org/archives/interview-creater-of-systemd-and-pulseaudio-lennart.html
