---
layout: post
title: "CentOS中双网卡静态路由配置"
description: "演示如何为双网卡配置独立的路由规则"
categories: [system]
tags: [centos, route]
---

>  一个网卡的话不需要静态路由的，如果多个网卡的话可以手工配置静态路由，特别是多个网卡走不同的子网的时候。

* Kramdown table of contents
{:toc .toc}

## 来自网上搜索的方法
之前一直没有配置过两个网卡分别使用不同的IP，走不同的网关，google了下发现了下面的手工添加路由的脚本：

~~~bash
#!/bin/sh
ip route add 10.1.1.0/24 dev br0 src 10.1.1.10 table bond0
ip route add default via 10.1.1.1 dev br0 table bond0
ip rule add from 10.1.1.10/32 table bond0
ip rule add to 10.1.1.10/32 table bond0

ip route add 192.168.1.0/24 dev br1 src 192.168.1.10 table bond1
ip route add default via 192.168.1.1 dev br1 table bond1
ip rule add from 192.168.1.10/32 table bond1
ip rule add to 192.168.1.10/32 table bond1
~~~

## 来自红帽文档中的方法
后来想了想这样的问题系统肯定已经支持得很好了，只是没有找到配置方法，于是找了下红帽的文档，发现可以像下面这样配置：

### 配置静态路由

 一个网卡的话不需要静态路由的，如果多个网卡的话可以手工配置静态路由，特别是多个网卡走不同的子网的时候。

`route -n   #查看当前路由信息`

静态路由配置文件路径:
`/etc/sysconfig/network-scripts/route-interface_name`
就和网卡的配置文件路径结构差不多，比如ifcfg-eth0变成了route-eth0。

eth0网卡的静态路由就保存在这个文件里面。这个文件可以有两种格式
- IP命令参数格式
- 网络/掩码指令格式

#### IP命令参数模式：

1）第一行定义默认路由：
```
default via X.X.X.X dev interface
```
X.X.X.X 是默认路由的IP. interface是可以连接到默认路由的网卡接口名.

2）静态路由一行一个：
```
X.X.X.X/X via X.X.X.X dev interface
```
X.X.X.X/X 是网络和掩码. X.X.X.X 和 interface 是各自网段的网关IP和网卡接口.

配置示例 route-eth0：
默认网关 192.168.0.1, 接口eth0. 两条静态路由到 10.10.10.0/24 和172.16.1.0/24 :

```
default via 192.168.0.1 dev eth0
10.10.10.0/24 via 10.10.10.1 dev eth1
172.16.1.0/24 via 192.168.0.1 dev eth0
```
#### 网络/掩码指令格式：

route-interface文件的第二种格式.下面是样板:
```
ADDRESS0=X.X.X.X
NETMASK0=X.X.X.X
GATEWAY0=X.X.X.X
```
ADDRESS0=X.X.X.X 静态路由的网络编号.
NETMASK0=X.X.X.X 为上面那行设置子网掩码 .
GATEWAY0=X.X.X.X  能够连接到 ADDRESS0=X.X.X.X 这个网络的网关

配置示例 route-eth0：
默认网关 192.168.0.1, 接口 eth0. 两条到10.10.10.0/24 和172.16.1.0/24 的静态路由：
```
ADDRESS0=10.10.10.0
NETMASK0=255.255.255.0
GATEWAY0=10.10.10.1
ADDRESS1=172.16.1.0
NETMASK1=255.255.255.0
GATEWAY1=192.168.0.1
```
ADDRESS0, ADDRESS1, ADDRESS2, 这样的编号必须是一个接一个的数字。
