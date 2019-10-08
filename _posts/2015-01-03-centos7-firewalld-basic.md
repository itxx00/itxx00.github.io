---
layout: post
title: "CENTOS7管理之动态防火墙FIREWALLD"
description: ""
categories: [system]
tags: [firewalld]
---

> firewalld 的出现带来了许多新的亮点，它使得防火墙规则管理更加方便和统一，但 firewalld 项目本身还有诸多不完善的地方，还需要一些时间的沉淀才能变得更加稳定和得到更多软件社区的支持。在我们使用过程中也发现了其存在的一些问题，我们也正在努力参与到 firewalld 的改进和测试当中，也希望有更多的人能够参与进来。

* Kramdown table of contents
{:toc .toc}

## firewalld的出现
自 fedora18 开始，firewalld 已经成为默认的防火墙管理组件被集成到系统中，用以取代 iptables service 服务，而基于 fedora18 开发的RHEL7，以及其一脉相承的CentOS7 也都使用 firewalld 作为默认的防火墙管理组件，无论是对于日常使用还是企业应用来讲这样的变更都或多或少的为使用者带来了影响。在浏览完本文前我们先不急着下结论到底这样的变化是好是坏，让我们“剥离它天生的骄傲，排除这些外界的干扰“，来看看 firewalld 到底是什么样的。
以下内容以 CentOS7 系统为例进行讲解，我们假设读者对于 iptables 防火墙相关知识有一定的掌握。

## 什么是动态防火墙？
我们首先需要弄明白的第一个问题是到底什么是动态防火墙。为了解答这个问题，我们先来回忆一下 iptables service 管理防火墙规则的模式：用户将新的防火墙规则添加进 /etc/sysconfig/iptables 配置文件当中，再执行命令 service iptables reload 使变更的规则生效。在这整个过程的背后，iptables service 首先对旧的防火墙规则进行了清空，然后重新完整地加载所有新的防火墙规则，而如果配置了需要 reload 内核模块的话，过程背后还会包含卸载和重新加载内核模块的动作，而不幸的是，这个动作很可能对运行中的系统产生额外的不良影响，特别是在网络非常繁忙的系统中。

如果我们把这种哪怕只修改一条规则也要进行所有规则的重新载入的模式称为静态防火墙的话，那么 firewalld 所提供的模式就可以叫做动态防火墙，它的出现就是为了解决这一问题，任何规则的变更都不需要对整个防火墙规则列表进行重新加载，只需要将变更部分保存并更新到运行中的 iptables 即可。

这里有必要说明一下 firewalld 和 iptables 之间的关系， firewalld 提供了一个 daemon 和 service，还有命令行和图形界面配置工具，它仅仅是替代了 iptables service 部分，其底层还是使用 iptables 作为防火墙规则管理入口。firewalld 使用 python 语言开发，在新版本中已经计划使用 c++ 重写 daemon 部分。

## firewalld 具有哪些特性？

那么 firewalld 除了是动态防火墙以外，它还具有哪些优势或者特性呢？第一个是配置文件。firewalld 的配置文件被放置在不同的 xml 文件当中，这使得对规则的维护变得更加容易和可读，有条理。相比于 iptables 的规则配置文件而言，这显然可以算作是一个进步。第二个是区域模型。firewalld 通过对 iptables 自定义链的使用，抽象出一个区域模型的概念，将原本十分灵活的自定义链统一成一套默认的标准使用规范和流程，使得防火墙在易用性和通用性上得到提升。令一个重要特性是对 ebtables 的支持，通过统一的接口来实现 ipt/ebt 的统一管理。还有一个重要特性是富语言。富语言风格的配置让规则管理变得更加人性化，学习门槛相比原生的 iptables 命令有所降低，让初学者可以在很短时间内掌握其基本用法，规则管理变得更快捷。

## firewalld 基本术语

本文不打算重复阐述现有文档中的基本知识，仅仅提供一些对现有文档中知识的补充和理解，详细的文档请参阅man page或本文结尾处的延伸阅读[^1]部分。

### zone：
firewalld将网卡对应到不同的区域（zone），zone 默认共有9个，block  dmz  drop  external  home  internal  public  trusted  work ，不同的区域之间的差异是其对待数据包的默认行为不同，根据区域名字我们可以很直观的知道该区域的特征，在CentOS7系统中，默认区域被设置为public，而在最新版本的fedora（fedora21）当中随着 server 版和 workstation 版的分化则添加了两个不同的自定义 zone FedoraServer 和 FedoraWorkstation 分别对应两个版本。使用下面的命令分别列出所有支持的 zone 和查看当前的默认 zone：
```
firewall-cmd --get-zones
firewall-cmd --get-default-zone
```

所有可用 zone 的 xml 配置文件被保存在 /usr/lib/firewalld/zones/ 目录，该目录中的配置为默认配置，不允许管理员手工修改，自定义 zone 配置需保存到 /etc/firewalld/zones/ 目录。防火墙规则即是通过 zone 配置文件进行组织管理，因此 zone 的配置文件功能类似于 /etc/sysconfig/iptables 文件，只不过根据不同的场景默认定义了不同的版本供选择使用，这就是 zone 的方便之处。

### service：
在 /usr/lib/firewalld/services/ 目录中，还保存了另外一类配置文件，每个文件对应一项具体的网络服务，如 ssh 服务等，与之对应的配置文件中记录了各项服务所使用的 tcp/udp 端口，在最新版本的 firewalld 中默认已经定义了 70+ 种服务供我们使用，当默认提供的服务不够用或者需要自定义某项服务的端口时，我们需要将 service 配置文件放置在 /etc/firewalld/services/ 目录中。service 配置的好处显而易见，第一，通过服务名字来管理规则更加人性化，第二，通过服务来组织端口分组的模式更加高效，如果一个服务使用了若干个网络端口，则服务的配置文件就相当于提供了到这些端口的规则管理的批量操作快捷方式。每加载一项 service 配置就意味着开放了对应的端口访问，使用下面的命令分别列出所有支持的 service 和查看当前 zone 种加载的 service：
```
firewall-cmd --get-services
firewall-cmd --list-services
```

## 使用示例

在 firewalld 官方文档中提供了若干使用示例，这些示例对学习防火墙管理是很好的基本参考资料。接下来我们将通过一些真实的使用示例来展示如何使用 firewalld 对防火墙规则进行管理。

### 场景一：自定义 ssh 端口号
出于安全因素我们往往需要对一些关键的网络服务默认端口号进行变更，如 ssh，ssh 的默认端口号是 22，通过查看防火墙规则可以发现默认是开放了 22 端口的访问的：

```
[root@localhost ~]# iptables -S
... ...
-A IN_public_allow -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

假设自定义的 ssh 端口号为 22022，使用下面的命令来添加新端口的防火墙规则：
```
firewall-cmd --add-port=22022/tcp
```

如果需要使规则保存到 zone 配置文件，则需要加参数 --permanent。我们还可以使用自定义 service 的方式来实现同样的效果：
在 `/etc/firewalld/services/` 目录中添加 自定义配置文件 custom-ssh.xml ，内容如下：

~~~xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>customized SSH</short>
  <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface, enable this option. You need the openssh-server package installed for this option to be useful.</description>
  <port protocol="tcp" port="22022"/>
</service>
~~~

执行命令重载配置文件，并添加防火墙规则:

~~~
systemctl reload firewalld
firewall-cmd --add-service=custom-ssh
~~~

一旦新的规则生效，旧的 ssh 端口规则旧可以被禁用掉：

```
firewall-cmd --remove-service=ssh
```

### 场景二：允许指定的IP访问SNMP服务
某些特殊的服务我们并不想开放给所有人访问，只需要开放给特定的IP地址即可，例如 SNMP 服务，我们将使用 firewalld 的富语言风格配置指令：

```
firewall-cmd --add-rich-rule="rule family='ipv4' source address='10.0.0.2' port port='161' protocol='udp' accept"
```

查看防火墙规则状态，证明结果正是我们想要的：

```
[root@localhost ~]# iptables -S
... ...
-A IN_public_allow -s 10.0.0.2/32 -p udp -m udp --dport 161 -m conntrack --ctstate NEW -j ACCEPT
```

参考链接：[^2]


[^1]: https://fedoraproject.org/wiki/FirewallD/zh-cn

[^2]: https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html

