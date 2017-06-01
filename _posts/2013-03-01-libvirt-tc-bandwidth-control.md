---
layout: post
title: "使用libvirt和tc实现vm带宽控制"
description: ""
categories: [virt]
tags: [libvirt, tc, bandwidth ]
---

> 在kvm虚拟机管理的过程当中，对虚拟机带宽进行良好的控制是十分重要的。

* Kramdown table of contents
{:toc .toc}


linux系统当中对网络带宽的控制一般都是使用tc命令实现，tc即是traffic control的缩写，在[这里](http://linux-ip.net/articles/Traffic-Control-HOWTO/)可以找到有关tc命令的内容。

当然你可以手动使用tc命令来处理这些事情，比如使用cbq队列，htb队列等，都是可以实现的，网上找找应该有很多关于这方面的资料，

## cbq队列示例
比如下面就是使用cbq队列限制src ip为192.168.1.102发送数据包的速率:

### 1.建立cbq队列:
```
tc qdisc add dev eth0 root handle 1: cbq avpkt 1000 bandwidth 100mbit
```
### 2.建立带宽限制分类:
```
tc class add dev eth0 parent 1: classid 1:1 cbq rate 60mbit allot 1500 prio 5 bounded isolated
tc class add dev eth0 parent 1: classid 1:2 cbq rate 70mbit allot 1500 prio 5 bounded isolated
tc class add dev eth0 parent 1: classid 1:3 cbq rate 80mbit allot 1500 prio 5 bounded isolated
```

### 3.建立过滤器，绑定指定带宽限制类型至指定虚拟机ip:
```
tc filter add dev eth0 parent 1: protocol ip prio 16 u32 match ip src 192.168.1.102 flowid 1:2
```

## htb队列示例
我们可以在母机上给vm对应的虚拟网卡增加tc规则，使用htb队列，一个可用的脚本示例如下：

~~~bash
# add interface bandwidth limit
# usage: tc_add iface in_kbps out_kbps
tc_add() {
    local iface=$1
    local in_bw=$2
    local out_bw=$3
    local in_average="${in_bw}kbps"
    local in_peak="${in_bw}kbps"
    local out_average="${out_bw}kbps"
    local out_peak="${out_bw}kbps"
    local burst="2kb"
    local mtu=1500
    local r2q=$((in_bw*1000/mtu-1))
    local tc="/sbin/tc"
    [ $r2q -lt 1 ] && r2q=1
    if [ $in_bw != 0 ]; then
        $tc qdisc add dev $iface root handle 1: htb default 2 r2q $r2q
        $tc class add dev $iface parent 1: classid 1:1 htb rate $in_average \
            ceil $in_peak burst $burst cburst $burst
        $tc class add dev $iface parent 1:1 classid 1:2 htb rate $in_average \
            ceil $in_peak burst $burst cburst $burst
        $tc qdisc add dev $iface parent 1:2 handle 2: sfq perturb 10
        $tc filter add dev $iface parent 1:0 protocol ip handle 1 fw flowid 1
    fi
    if [ $out_bw != 0 ]; then
        $tc qdisc add dev $iface ingress
        $tc filter add dev $iface parent ffff: protocol ip u32 match ip src \
            0.0.0.0/0 police rate $out_average burst ${out_bw}kb mtu 64kb drop \
            flowid :1
    fi
}

# clean up interface bandwidth limit
# usage: tc_del iface
tc_del() {
    local iface=$1
    local tc="/sbin/tc"
    $tc qdisc del dev $iface root &>>/dev/null
    sleep 0.1
    $tc qdisc del dev $iface ingress &>>/dev/null
}

~~~

## libvirt中的带宽控制
我比较推荐的方法还是直接使用libvirt，libvirt 中已经集成了带宽控制的功能，下面是关于带宽控制部分的xml描述:

使用方法：在网卡interface中加入
~~~xml
<bandwidth>
<inbound average='1000' peak='5000' burst='1024'/>
<outbound average='128' peak='256' burst='256'/>
</bandwidth>
~~~

以下是关于各项参数的解释，获取最新的信息可以参考[libvirt文档](http://www.libvirt.org/).

* mandatory attribute:
  * average: It specifies average bit rate on interface being shaped.

* optional attributes:
  * peak: which specifies maximum rate at which interface can send data,
  * burst: amount of bytes that can be burst at peak speed.

Accepted values: integer numbers.

units:
* average: kilobytes per second
* peak: kilobytes per second
* burst: kilobytes.
