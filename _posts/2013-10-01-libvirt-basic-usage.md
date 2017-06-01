---
layout: post
title: "libvirt中的网络管理实践"
description: ""
categories: [virt]
tags: [libvirt, network]
---

> 逐步分析libvirt中的网络管理方法及实践，分析在nat网络中遇到的问题及解决思路

* Kramdown table of contents
{:toc .toc}



## libvirt网络基本概念

libvirt默认使用了一个名为default的nat网络，这个网络默认使用virbr0作为桥接接口，使用dnsmasq来为使用nat网络的虚拟机提供dns及dhcp服务，dnsmasq生效后的配置文件默认保存在以下路径：

- /var/lib/libvirt/dnsmasq/default.hostsfile   mac&&ip绑定的配置文件
- /var/lib/libvirt/dnsmasq/default.leases  dhcp分配到虚拟机的ip地址列表
- /var/lib/libvirt/network/default.xml  default网络的配置文件

dnsmasq服务的启动脚本在/etc/init.d/dnsmasq ，但是我们如果手动使用此脚本来启动服务将会导致dnsmasq读取其自己的配置文件来启动此服务，因此这么做是不推荐的，因为这个服务完全由libvirtd在接管，当libvirtd服务启动的时候，它会将它管理的被标记为autostart的network一并启动起来，而启动network的时候就会自动调用dnsmasq并赋予其适宜的配置文件来运行服务。

使用libvirt管理的网络都会用到dnsmasq来产生相应的配置，比如定义了一个名为route110的network，那么这个route110将使用一个新的桥接接口virbr1来接入网络，并使用dnsmasq产生名为route110.hostsfile和route110.leases的配置文件。其实这里提到的virbr0和virbr1都是libvirt产生的虚拟网卡，其作用就相当于一个虚拟交换机，为虚拟机提供网络转发服务。


## libvirt中网络的类型

首先分析一下libvirt所能提供的网络类型：isolated 和forwarding,其中，isolated意为绝对隔离的网络，也就是说处于此网络内的虚拟机对于外界是隔离的，这种模式可以用到一些特殊的场合，比如虚拟机只提供给内部使用，虚拟机只要求能相互通信而不需要与互联网通信。另外一类，forwarding，就是把虚拟机的数据forward到物理网络实现与外部网络进行通讯，其中forwarding又分为两种：nat和routed。

### nat
就是把虚拟机的网络数据在经过物理机网络的时候进行ip伪装，这样所有虚拟机出去的网络数据都相当于是物理机出去的数据，也就是说，我们可以分配给使用nat网络的虚拟机一个内网ip，而这个内网ip的虚拟机访问出去的时候外部网络看到的是物理机的公网ip，这样做的用处就是实现多个虚拟机共享物理主机的公网ip，节省公网ip地址；如前所述，默认情况下libvirt已经提供了一个名为default的nat网络，在不需要进行任何配置的情况下使用default网络的虚拟机即可访问互联网，但是互联网却无法访问虚拟机提供的服务，这是因为default网络只对虚拟机的数据包进行了伪装，而没有进行dnat和snat。

需要注意的是libvirt所实现的这种nat网络是通过物理机的iptables规则来实现的，也即是在虚拟机数据经过nat表的postrouting链出去的时候对其进行了伪装。

### routed
forwarding模式的另外一种，routed，就是将虚拟机的数据直接通过物理机route出去，和nat一样，也是需要一个virbr虚拟网卡接口来与外面进行通信，这种模式的不同之处在于虚拟机的数据没有经过伪装便直接交给了外部网络，也就是说，使用route模式网络的虚拟机可以使用公网ip地址，而物理机却恰恰在这个时候完全可以使用一个内网ip而不对外提供访问，这样，虚拟机的网卡仅仅把物理机当作一个route数据的工具，此模式应用的场合很多，比如需要让虚拟机运行在一个dmz网络中。但是使用route模式有诸多限制，例如物理机的网络接口不够用的情况下。

这里需要注意的是，nat模式和route模式的区别仅仅在于前者使用了iptables对虚拟机的数据包进行了伪装，而后者没有。


## 自定义routed网络

在实际的虚拟机使用过程中，我们可能会碰到下面的情况：

- 1 使用nat网络的虚拟机也需要对外提供服务，
- 2 物理机只有一个网卡和一个ip，而我们现在既需要通过这个网卡来管理虚拟机，又需要使用这个网卡来提供route网络。

当然你所能碰到的问题可能千奇百怪，也可能根本没有碰到过此类bt问题。下面的内容只作为分析和解决问题的思路，不能生搬。在了解了libvirt的网络管理模式之后，就可以自己动手解决这些限制，下面重点解释第二种问题的解决方法：

首先假定route网络使用的是virbr1虚拟网卡，而虚拟机使用virbr1来为虚拟机提供服务，而我本机又有了一个br0作为em1的桥接网卡来对外提供网络服务，br0的ip是192.168.1.51

首先禁用br0：

~~~
ifdown br0
~~~

并配置br0的onboot为no,配置文件为`onboot=no`

然后我们定义了一个名为route的网络，virbr1的ip设置为192.168.1.51 ，这样做的目的是让virbr1取代之前的br0.

~~~xml
<network>
<name>route</name>
<uuid>6224b437-386b-f510-11d5-58d58b1ce87a</uuid>
<forward mode='route'/>
<bridge name='virbr1' stp='on' delay='0' />
<mac address='52:54:00:C8:9F:07'/>
<ip address='192.168.1.51' netmask='255.255.255.0'>
<dhcp>
<range start='192.168.1.128' end='192.168.1.254' />
</dhcp>
</ip>
</network>
~~~

接着生成并启用该网络
~~~
virsh net-define route.xml
virsh net-start route
virsh net-autostart route
~~~

- /etc/libvirt/qemu/networks/  virsh net-define的network会保存到这
- /var/lib/libvirt/network/  net-start启动了的network同时也会会保存到这
- /etc/libvirt/qemu/networks/autostart/  net-autostart的network同时也会保存到这

接下来，我们需要修改em1的配置并将其桥接到virbr1上

ifcfg-em1

~~~
DEVICE="em1"
ONBOOT="yes"
BRIDGE=virbr1
~~~

接着启动em1

~~~
ifup em1
~~~

至此em1就被桥接到了virbr1上，可以使用下面的命令检查

~~~
brctl show
~~~

现在我们需要在本机添加一条默认路由，不然虚拟机是访问不了外面的：

~~~
route add default gw 192.168.1.1 dev virbr1
~~~

这里的192.168.1.1是真实的路由。至此，问题已经解决了。

## 自定义nat网络

下面说说问题1的解决方法：
既然知道了nat出去的虚拟机只能访问外网而外网却不能访问进来，nat又是通过iptables来做的，也就是当libvirt每次启动的时候都会往iptables最前面插入自己的规则以保证nat的虚拟机能正常访问外网，那么我们是不是可以通过修改iptables的规则来实现呢，比如我们需要一个内网ip的虚拟机对外提供80服务，那么我们就把物理机的80端口映射到这台虚拟机的80端口上，因为我们的物理机是可以直接和虚拟机通信的，只是外网不能而已，下面添加规则：

~~~
iptables -t nat -A PREROUTING -p tcp -i virbr1 --dport 80  -j DNAT --to-destination 192.168.122.2:80
~~~

这样我们对外部访问80端口进来的数据进行了dnat，而出去的我们不用snat，只需要再添加如下规则：

~~~
iptables -I FORWARD -i virbr1 -o virbr0 -p tcp -m state --state NEW -j ACCEPT
~~~

至此问题看似得到解决，但是我们忽略了一个关键的问题，那就是每当libvirt启动的时候就会往表的最前面插入它自己的规则，而iptables的规则是有先后顺序的，也就是说，我们自己添加的规则在libvirtd服务重启之后即被libvirt定义的规则所淹没，怎么办呢，我现在只想到了这么一个方法，直接修改libvirtd的启动脚本，在它的规则生效之后插入我们自定义的规则：

vi  /etc/init.d/libvirtd
~~~
start() {
    echo -n $"Starting $SERVICE daemon: "
    initctl_check

    mkdir -p /var/cache/libvirt
    rm -rf /var/cache/libvirt/*
    KRB5_KTNAME=$KRB5_KTNAME daemon --pidfile $PIDFILE --check $SERVICE $PROCESS --daemon $LIBVIRTD_CONFIG_ARGS $LIBVIRTD_ARGS
    RETVAL=$?
    echo
    [ $RETVAL -eq 0 ] && touch /var/lock/subsys/$SERVICE
    sleep 1
    iptables -D FORWARD -i virbr1 -o virbr0 -p tcp -m state --state NEW -j ACCEPT
    iptables -I FORWARD -i virbr1 -o virbr0 -p tcp -m state --state NEW -j ACCEPT
... ...
~~~

至此问题基本解决。

## route网络转换nat网络
另外一个问题，我们前面有发现route和nat的网络区别仅仅是一个做了nat的iptables规则一个没有，那么我们可不可以自己在iptables里面添加相应的规则将route网络变身为nat网络呢？答案肯定是可以的，只需要添加上下面的规则即可,原理还请观看本文的同学自己分析，这里假设我们route网络给虚拟机分配的ip是192.168.100.0/24网段：

~~~
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -d ! 192.168.100.0/24 -j MASQUERADE
iptables -A FORWARD --destination 192.168.100.0/24 -m state --state RELATED,ESTABLISHED -j ACCEPT
~~~

## 自定义dnsmasq
这里再添加一个可以手工启动dnsmasq的小脚本

~~~bash
#!/bin/bash
brctl addbr routebr
ifconfig routebr 192.168.122.1 netmask 255.255.255.0
iptables -t nat -A POSTROUTING -s 192.168.122.0/24 -d ! 192.168.122.0/24 -j MASQUERADE
iptables -A FORWARD --destination 192.168.122.0/24 -m state --state RELATED,ESTABLISHED -j ACCEPT
/usr/sbin/dnsmasq \
--strict-order \
--bind-interfaces \
--pid-file=/usr/local/vps/network/default.pid \
--conf-file= \
--except-interface lo \
--listen-address 192.168.122.1 \
--dhcp-range 192.168.122.2,192.168.122.254 \
--dhcp-leasefile=/usr/local/vps/network/dnsmasq/default.leases \
--dhcp-lease-max=253 \
--dhcp-no-override \
--dhcp-hostsfile=/usr/local/vps/network/dnsmasq/default.hostsfile
~~~

## 重启network导致网络中断

当我们需要实时修改network的配置并使之生效的时候，就得重新启动此network，也就是需要net-destroy再net-start一下，我们的配置才能生效，但是随之而来的问题是，当network被重新启动之后，虚拟机便无法访问网络了，除非把虚拟机的network interface重新attach一下，或者等到虚拟机重新启动，那么为什么会出现这样的问题呢？我们先从它的表象开始分析，至于是否要追究到源码里面就取决于同学们自己了，反正我暂时没那功夫。这里仅仅是抛出来了一块砖。

当一个network启动之后，会自动生成一个虚拟网卡接口如virbr1，也会生成其他一些需要的东西，而重新启动了libvirt的network之后这个接口也会被重启，所以就导致了中途有一个中断的过程，

那事情就比较清晰了，如果你将libvirt启动网络的所有过程拆分开来一个一个的手动生成，需要修改某一部分配置的时候实际上你只需要修改对应的配置文件而不需要重新启动这个virbr1接口，比如上面提到的mac+ip的绑定，如果把dnsmasq独立出来，不让libvirt接管，那么增加了mac+ip绑定之后，仅仅需要重启dnsmasq这个服务。

