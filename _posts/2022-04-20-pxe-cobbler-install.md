---
layout: post
title: "标题"
description: "描述"
categories: [shell]
tags: [bash, ]
---


* Kramdown table of contents
{:toc .toc}


### 基本介绍：
PXE（preboot execute environment）由Intel发明的通过网络快速引导操作系统的技术，其原理是在机器引导时通过server端为网卡DHCP分配IP信息，并通知client端next_server中的tftp地址，client端继续通过tftp下载系统引导镜像，加载并完成启动。这里我们还会用到另外一项技术叫kickstart，由红帽开发，早先用于其系统安装工具中以完成自动化安装，已被众多发行版支持。系统引导时可以通过kickstart配置文件中指定的安装流程自动完成后续步骤，减少人工干预。而通常手工配置dhcp、tftp、kickstart等往往比较繁琐，这里我们会利用红帽开发的另外一款工具cobbler，通过cobbler来完成整个dhcp、tftp、kickstart等组成的server端环境的快速搭建和管理，以此提高效率。

### cobbler安装配置：
我们使用CentOS7作为server端系统，为了节约现场部署时间，我们将提前准备好环境并直接带到现场使用，以下所有操作将在一台ThinkPad上完成。

因私有化环境无需连外网，因此在实际使用时我们为了简化部署流程，可以将selinux和防火墙禁用掉，如需要启用防火墙的话则需要放开http/dhcp/tftp等服务的对应端口：

```
# disable selinux
sed -i 's/^SELINUX=.*$/SELINUX=disabled/' /etc/selinux/config

# disable iptables
systemctl disable firewalld
systemctl stop firewalld

reboot
```

 安装cobbler及相关的依赖包：cobbler提供了命令行管理工具和一个web管理工具，分别由cobbler和cobbler-web两个包提供
```
yum install epel-release
yum install cobbler cobbler-web httpd dhcp tftp xinetd rsync bind
```

配置cobbler：cobbler配置文件放置在/etc/cobbler目录，在启动之前需要server端IP，dhcp等相关信息，首先修改 /etc/cobbler/settings主配置文件，需要修改的参数有以下：
```
# 通过以下命令生成系统安装后的默认root密码
openssl passwd -1
# 并将生成的密码修改到配置中
default_password_crypted: “$1$RUNYOYnz$QgzdhCD2T7qXWI1IPpAih0”

# server端ip，对外提供dhcp和http服务，必须为一个固定内网ip地址
server: 192.168.1.1

# next_server为tftp服务所在ip，通常是需要和server保持一致
next_server: 192.168.1.1

# 打开cobbler对相关服务的自动管理功能，如配置变更和启停等
manage_dhcp: 1
manage_tftpd：1
```

 修改依赖组件的配置：
```
sed -i '/disable/c\\tdisable\t\t\t= no' /etc/xinetd.d/tftp
service xinetd restart
修改dhcp网段：vi /etc/cobbler/dhcp.template
subnet 192.168.1.0 netmask 255.255.255.0 {
     option routers             192.168.1.1;
     option domain-name-servers 192.168.1.1;
     range dynamic-bootp        192.168.1.100 192.168.1.200;
     option subnet-mask         255.255.255.0;
     filename                   "/pxelinux.0";
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                $next_server;
}
```

启动服务：
```
systemctl start httpd
systemctl start cobblerd

systemctl enable httpd
systemctl enable cobblerd
 服务检查：cobbler提供了check命令可用于检查各项配置是否满足需要

cobbler check
# 通常第一次会提示下载loader
cobbler get-loaders
# 如中途修改cobbler配置后需重启cobbler服务
systemctl restart cobblerd
# 如变更了dhcp、tftp等相关信息需重新同步配置
cobbler sync
# 顺便配置好web管理页面的访问密码
htdigest /etc/cobbler/users.digest "Cobbler" cobbler
```

 可以反复通过check命令来检查环境是否部署OK，并根据实际需求调整各项配置文件，直至check结果复合要求即可。至此cobbler的安装及配置完成。web端工具访问地址：https://192.168.1.1/cobbler_web



### 系统镜像准备：

接下来我们需要将系统镜像导入cobbler中，并自定义安装引导的kickstart配置。我们要部署到节点上的系统是CentOS7。需要注意的是如果需要通过kickstart定制一些基础软件包的安装，那么需要使用软件包更全的DVD iso，因minimal iso中提供的软件包有限。
```
# 将iso挂载到本地目录
mount -o auto CentOS-7-x86_64-DVD-1611.iso /mnt/
# 导入到cobbler中
cobbler import --name=centos7 --arch=x86_64 --path=/mnt
# 查看导入的系统及profile
cobbler distro list
cobbler distro report --name=centos7-x86_64
cobbler profile list
# 卸载iso mount point
umount /mnt/
```

 可以看到上面的步骤中我们将CentOS7镜像导入到cobbler中，有几个核心概念需要理解：

distro - 及系统发行版本，不同的镜像导入后对应不同的distro，如centos7-x86_64，不同的distro对应不同的引导镜像；

profile - distro的配置文件，一个distro可以有多个profile，默认导入时会自动生成一个profile，不同的profile可以定义不同的kernel选项，使用不同的kickstart配置；

system - 各个机器所使用的profile实例，与机器MAC地址绑定，可以细化到机器级别的自定义安装，如果所有机器安装都是统一的则无需使用system配置。

 接下来需要理解的是cobbler中对kickstart文件的管理方式，ks文件是我们需要重点关注的中间产物，决定了系统自动化部署的执行流程和最终效果。ks文件与profile绑定，默认生成的profile会指向一个默认的ks文件，通常我们需要对其进行自定义来满足不同的部署要求。当系统通过PXE引导至profile选择菜单后，一旦选定了需要部署的系统，接下来就会按照该profile所对应的ks文件来执行一系列的安装操作。

在cobbler中ks文件的实例是通过cgi动态生成的，而生成ks实例所依赖的则是ks templates和snippets， cobbler通过template来将ks文件主体流程部分模板化，通过snippets来管理可以在不同ks templates中公用的流程片段。

我们的需求如下：

安装一个精简的CentOS7系统；
同时默认安装一些必要的软件包；
首次安装时只对系统盘进行分区和格式化，其他磁盘不动；
为了便于管理我们将更改网卡名为ethX，且默认禁用IPv6,；
为了方便使用虚拟机测试整个安装流程，需要在磁盘分区时自动适配磁盘名如vda/sda；
安装完成后能对一些基础配置进行初始化。       

首先拷贝cobbler默认的template生成一个自定义的ks template，
```
# kickstart template for TBDS
# (includes %end blocks)
# do not use with earlier distros

#platform=x86, AMD64, or Intel EM64T
# System authorization information
auth --useshadow --enablemd5
# System bootloader configuration
#bootloader --location=mbr
# Partition clearing information
clearpart --all --initlabel
# Use text mode install
text
# Firewall configuration
firewall --disabled
# Run the Setup Agent on first boot
firstboot --disable
# System keyboard
keyboard us
# System language
lang en_US
# Use network installation
url --url=$tree
# If any cobbler repo definitions were referenced in the kickstart profile, include them here.
$yum_repo_stanza
# Network information
$SNIPPET('network_config')
# Reboot after installation
reboot

#Root password
rootpw --iscrypted $default_password_crypted
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx
# System timezone
timezone Asia/Shanghai

# Install OS instead of upgrade
install
# Clear the Master Boot Record
zerombr
# Allow anaconda to partition the system as needed
#autopart
$SNIPPET('main_partition_select')

%pre
$SNIPPET('log_ks_pre')
$SNIPPET('kickstart_start')
$SNIPPET('pre_install_network_config')
$SNIPPET('pre_partition_select_tbds')
# Enable installation monitoring
$SNIPPET('pre_anamon')
%end

%packages
@^minimal
@core
chrony
wget
net-tools
python-setuptools
rsync
lrzsz
expect
tcl
ntpdate
-selinux-policy*
-NetworkManager*
-kexec-tools
-snappy
-wpa_supplicant
-ppp
%end

%addon com_redhat_kdump --disable --reserve-mb='auto'

%end

%post --nochroot
$SNIPPET('log_ks_post_nochroot')
%end

%post
$SNIPPET('log_ks_post')
# Start yum configuration
$yum_config_stanza
# End yum configuration
$SNIPPET('post_install_kernel_options')
$SNIPPET('post_install_network_config')
$SNIPPET('download_config_files')
$SNIPPET('cobbler_register')
# Enable post-install boot notification
$SNIPPET('post_anamon')
$SNIPPET('post_install_custom_sys')
# Start final steps
$SNIPPET('kickstart_done')
# End final steps

%end
```

 注意ks template中的红色部分为我们增加的自定义snippets，第一个pre_partition_select_tbds作用是自动根据磁盘类型来生成分区和格式化选项，同时兼容虚拟机和物理机，内容如下：

```
# Determine architecture-specific partitioning needs
if [ -b /dev/vda ]; then
  cat >/tmp/partinfo << EOF
clearpart --initlabel --all
ignoredisk --only-use=vda
bootloader --location=mbr --boot-drive=vda --driveorder=vda
clearpart --initlabel --drives=vda
part /boot --fstype=ext3 --ondisk=vda --size=500
part / --fstype=xfs --size=1024 --grow --ondisk=vda --asprimary
EOF
elif [ -b /dev/sda ]; then
  cat >/tmp/partinfo << EOF
clearpart --initlabel --all
ignoredisk --only-use=sda
bootloader --location=mbr --boot-drive=sda --driveorder=sda
part /boot --fstype=ext3 --ondisk=sda --size=500
part / --fstype=xfs --size=100000 --ondisk=sda --asprimary
part /data --fstype=xfs --grow --ondisk=sda
EOF
fi
``` 

 第二个post_install_custom_sys作用是在系统安装最后阶段对一些必要的配置进行更改，其中运行的是shell脚本，内容如下：

```
# cat snippets/post_install_custom_sys
if ! grep -q 'tbds_customize' /etc/sysctl.conf; then
  cat >>/etc/sysctl.conf<<EOF
## tbds_customize
fs.file-max = 262144
net.core.somaxconn = 10240
vm.swappiness = 0
net.ipv4.ip_local_port_range = 1024 65000
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 1048576
net.core.wmem_default = 524288
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.netdev_max_backlog = 2500
net.ipv4.tcp_max_syn_backlog = 40960
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
EOF
fi

chmod +x /etc/rc.d/rc.local

if grep -q '^UseDNS' /etc/ssh/sshd_config; then
  sed -i 's/^UseDNS .*/UseDNS no/' /etc/ssh/sshd_config
else
  sed -i 's/^#UseDNS .*/UseDNS no/' /etc/ssh/sshd_config
fi
 ```


接下来还需要修改内核引导参数，完成网卡名字的变更及IPv6禁用：



通过这几部分的组合，即可生成一个完整可用的ks文件，下面我将介绍如何通过虚拟机来测试安装。

### 使用虚拟机测试PXE：
安装虚拟化相关软件包，使用kvm虚拟机，同时安装图形界面虚拟机管理工具virt-manager方便安装操作。网络选择使用bridge模式,点击新建虚拟机，在安装选项中选择PXE,注意内存设置必须大于1G，否则PXE引导进入系统后很可能报错。
