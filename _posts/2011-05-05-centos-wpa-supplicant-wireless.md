---
layout: post
title: "使用wpa-supplicant配置无线网络"
description: ""
categories: [system]
tags: [wpa-supplicant, wireless ]
---

> 之前在使用桌面的过程中，发现如果需要连接无线网 ， 那么networkmanager是首选的，自动管理网卡，自动扫描信号，用起来各种舒服；但是突然有一天发现networkmanager干了某些我不期望它干的事情，于是果断yum remove之，由于一气之下remove掉了networkmanager，并没有考虑到还得用它来连接无线，导致后来发现需要连无线的时候极为不方便，最终发现可以使用wpa-supplicant来管理无线网络连接。

* Kramdown table of contents
{:toc .toc}


其实很多人都推荐用这个了wpa_supplicant，但是由于一直没有这个必要所以就一直没有学会使用，今天用了下感觉还真不错。 wpa_supplicant首先在/etc/init.d/wpa_supplicant 有一个启动控制脚本，然后有/etc/wpa_supplicant/wpa_supplicant.conf这个默认配置文件，还有一个/etc/sysconfig/wpa_supplicant的全局配置文件，

### 安装
使用yum安装：`yum install wpa_supplicant`，在/usr/share/doc下有大量的配置示例和文档，

### 配置
在使用它之前需要修改几个配置文件，首先是wpa_supplicant.conf  ,比如我的配置如下:

```
# cat /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
network={
ssid="yourssid"
#scan_ssid=1
proto=WPA2
key_mgmt=WPA-PSK
pairwise=CCMP
group=CCMP
psk="fightandfuck"
priority=2
}
```

关于这些信息可以先使用`iwlist  wlan0 scan` 命令扫描，并做相应调整。然后需要修改全局配置文件

cat /etc/sysconfig/wpa_supplicant

```
INTERFACES="-iwlan0"
DRIVERS="-Dwext"
OTHER_ARGS="-f /var/log/wpa_supplicant.log -P /var/run/wpa_supplicant.pid"
```
### 启动
修改完这些后，即可使用`service wpa_supplicant start` 启动网卡并连接认证了，当然可能出现的情况是认证成功了但是网卡没有获取到ip地址，这个时候只需要手动dhclient一下或者写个ifcfg-wlan0并指定其使用dhcp然后ifup wlan0。

总之，方便适用。
