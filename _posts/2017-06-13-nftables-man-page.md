---
layout: post
title: "nftables：nft man文档阅读笔记"
description: "这里是一份nft man文档笔记"
categories: [system]
tags: [nft, nftables ]
---

> 利用空闲时间学习了nftables的基础知识，其中官方的man page中包含了大量信息，在阅读过程中整理了一份带中文注释的笔记，以辅助加深记忆。

* Kramdown table of contents
{:toc .toc}


## 工具名称

nft --  包过滤规则管理工具



## 基本用法

`nft [ -n | --numeric ] [ -s | --stateless ] [ [-I | --includepath] directory ] [ [-f | --file] filename | [-i | --interactive] | cmd ]`

`nft [ -h | --help ] [ -v | --version ]`


## 工具描述

nftables 作为新一代的防火墙策略框架，旨在替代之前的各种防火墙工具诸如iptables/ebtables等，而且提供了类似tc的带宽限速能力。而nft则提供了nftables的命令行入口，是用户空间的管理工具。



## 选项说明

执行`nft --help`查看完整帮助信息



`-h, --help`

<dd>


查看帮助信息.

</dd>

`-v, --version`

<dd>


查看版本号.

</dd>

`-n, --numeric`

<dd>

以数值方式展示数据，可重复使用，一个-n表示不解析域名，第二次不解析端口号，第三次不解析协议和uid/gid。

</dd>

`-s, --stateless`

<dd>

省略规则和有状态对象的状态信息
</dd>

`-N`

<dd>

将ip地址解析成域名，依赖dns解析。

</dd>

`-a, --handle`

<dd>

输出内容中展示规则handle信息

</dd>

`-I, --includepath directory`

<dd>

添加include文件搜索目录

</dd>

`-f, --file filename`

<dd>

从文件获取输入

</dd>

`-i, --interactive`

<dd>

从交互式cli获取输入

</dd>



## 文件格式


### 语法规定

单行过长可用`\`换行连接；
多个命令写到同一行可用分号`;` 分隔；
注释使用井号`#`打头；
标识符用大小写字母打头，后面跟数字字母下划线正斜杠反斜杠以及点号;
用双引号引起来表示纯字符串。

### 文件引用

`include "filename"`

可由外部文件通过`include`导入到当前文件，用`-I/--includepath`指定导入文件所在目录，如果include后面接的是目录而非文件，则整个目录的文件将以字母顺序依次导入。



### 符号变量

`define variable = expr`

`$variable`

Symbolic variables can be defined using the ***define*** statement. Variable references are expressions and can be used initialize other variables. The scope of a definition is the current block and all blocks contained within.
变量使用`define`定义，变量引用属于表达式，可以用于初始化其他变量，变量的生效范围在当前block以及被包含的所有block内。


***示例 1. 使用符号变量***

~~~
define int_if1 = eth0
define int_if2 = eth1
define int_ifs = { $int_if1, $int_if2 }

filter input iif $int_ifs accept
~~~


## 地址族

根据处理的包的种类不同可以将其分为不同的地址族。不同的地址族在内核中包含有特定阶段的处理路径和hook点，当对应hook的规则存在时则会被nftables处理。具体类型如下：


`ip`

<dd>

IPv4 地址族

</dd>

`ip6`

<dd>

IPv6 地址族

</dd>

`inet`

<dd>

Internet (IPv4/IPv6) 地址族

</dd>

`arp`

<dd>

ARP 地址族

</dd>

`bridge`

<dd>

Bridge 地址族

</dd>

`netdev`

<dd>

Netdev 地址族

</dd>


所有nftables对象存在于特定的地址族namespace中，换言之所有identifier都含有一个特定的地址族，如果未指定则默认使用`ip`地址族
### IPv4/IPv6/Inet address families

IPv4/IPv6/Inet 地址族用于处理 IPv4和IPv6包，其在network stack中在不同的包处理阶段一共包含了5个hook.

***Table 1. IPv4/IPv6/Inet 地址类hook列表***

| Hook名称 | 描述 |
| --- | --- |
| prerouting | 所有进入到系统的包都会被prerouting hook进行处理. 它在routing流程之前就被发起，用于靠前阶段的包过滤或者更改影响routing的包属性. |
| input | 发往本地系统的包将被input hook处理. |
| forward | 被转发到其他主机的包会经由forward hook处理. |
| output | 由本地进程发送出去的包将被output hook处理. |
| postrouting | 所有离开系统的包都将被postrouting hook处理. |



### ARP address family

ARP地址族用于处理经由系统接收和发送的ARP包。一般在集群环境中对ARP包进行mangle处理以支持clustering。


***Table 2. ARP address family hooks***

| Hook | 描述 |
| --- | --- |
| input | 分发到本机的包会经过input hook. |
| output | 由本机发出的包会经过output hook. |



### Bridge address family

bridge地址族处理通过桥接设备的ethernet包。



### Netdev address family

Netdev地址族处理从ingress过来的包。


***Table 3. Netdev address family hooks***

| Hook | Description |
| --- | --- |
| ingress | 所有进入系统的包都将被ingress hook处理。它在进入layer 3之前的阶段就开始处理。|



## Tables

`{add | delete | list | flush} table [family] {table}`

table是chain/set/stateful object的容器，table由其地址族和名字做标识。地址族必须属于ip, ip6, arp, bridge, netdev中的一种，inet地址族是一个虚拟地址族，同来创建同时包含IPv4和IPv6的table，如果没有指定地址族则默认使用`ip`地址族。


`add`

<dd>

添加指定地址族，指定名称的table

</dd>

`delete`

<dd>

删除指定的table

</dd>

`list`

<dd>

列出指定table中的所有chain和rule

</dd>

`flush`

<dd>

清除指定table中的所有chain和rule

</dd>



## Chains

`{add} chain [family] {table} {chain} {hook} {priority} {policy} {device}`

`{add | create | delete | list | flush} chain [family] {table} {chain}`

`{rename} chain [family] {table} {chain} {newname}`

chain是rule的容器，他们存在于两种类型，基础链（base chain）和常规链（regular chain）。base chain是网络栈中数据包的入口点，regular chain则可用于jump的目标并对规则进行更好地组织。


`add`

<dd>

在指定table中添加新的链，当hook和权重值被指定时，添加的chain为base chain，将在网络栈中hook相关联。

</dd>

`create`

<dd>

与`add`命令类似，不同之处在于当创建的chain存在时会返回错误。

</dd>

`delete`

<dd>

删除指定的chain，被删除的chain不能有规则且不能是跳转目标chain。

</dd>

`rename`

<dd>

重命名chain

</dd>

`list`

<dd>

列出指定chain中的所有rule

</dd>

`flush`

<dd>

清除指定chain中所有rule

</dd>


## Rules

`[add | insert] rule [family] {table} {chain} [position position] {statement...}`

`{delete} rule [family] {table} {chain} {handle handle}`

Rules are constructed from two kinds of components according to a set of grammatical rules: expressions and statements.


`add`

<dd>

Add a new rule described by the list of statements. The rule is appended to the given chain unless a position is specified, in which case the rule is appended to the rule given by the position.

</dd>

`insert`

<dd>

Similar to the ***add*** command, but the rule is prepended to the beginning of the chain or before the rule at the given position.

</dd>

`delete`

<dd>

Delete the specified rule.

</dd>


## Sets

`{add} set family] {table} {set}{ {type} [flags] [timeout] [gc-interval] [elements] [size] [policy]}`

`{delete | list | flush} set [family] {table} {set}`

`{add | delete} element [family] {table} {set}{ {elements}}`

Sets are elements containers of an user-defined data type, they are uniquely identified by an user-defined name and attached to tables.
sets 是用户定义的数据类型的容器，具有用户定义的唯一标识，被应用到table上。


`add`

<dd>

在指定的table中添加一个新的set

</dd>

`delete`

<dd>

删除指定set

</dd>

`list`

<dd>

查看set内的元素
</dd>

`flush`

<dd>

清空整个set

</dd>

`add element`

<dd>

往set中添加元素，多个使用逗号分隔

</dd>

`delete element`

<dd>

从set中删除元素，多个使用逗号分隔

</dd>



***Table 4. Set 参数***

| 关键字 | 描述 | 类型 |
| --- | --- | --- |
| type | 元素的数据类型 | string: ipv4_addr, ipv6_addr, ether_addr, inet_proto, inet_service, mark |
| flags | set flags | string: constant, interval, timeout |
| timeout | 元素在set中的存活时间 | string, 带单位的小数. 单位: d, h, m, s |
| gc-interval | 垃圾回收间隔, 仅当timeout或flag timeout设置时生效 | string, decimal followed by unit. Units are: d, h, m, s |
| elements | set中包含的元素 | set data type |
| size | set可存放的最大元素个数 | unsigned integer (64 bit) |
| policy | set policy | string: performance [default], memory |



## Maps

`{add} map [family] {table} {map}{ {type} [flags] [elements] [size] [policy]}`

`{delete | list | flush} map [family] {table} {map}`

`{add | delete} element [family] {table} {map}{ {elements}}`

Maps store data based on some specific key used as input, they are uniquely identified by an user-defined name and attached to tables.


`add`

<dd>

Add a new map in the specified table.

</dd>

`delete`

<dd>

Delete the specified map.

</dd>

`list`

<dd>

Display the elements in the specified map.

</dd>

`flush`

<dd>

Remove all elements from the specified map.

</dd>

`add element`

<dd>

Comma-separated list of elements to add into the specified map.

</dd>

`delete element`

<dd>

Comma-separated list of element keys to delete from the specified map.

</dd>




***Table 5. Map specifications***

| Keyword | Description | Type |
| --- | --- | --- |
| type | data type of map elements | string ':' string: ipv4_addr, ipv6_addr, ether_addr, inet_proto, inet_service, mark, counter, quota. Counter and quota can't be used as keys |
| flags | map flags | string: constant, interval |
| elements | elements contained by the map | map data type |
| size | maximun number of elements in the map | unsigned integer (64 bit) |
| policy | map policy | string: performance [default], memory |



## Stateful objects

`{add | delete | list | reset} type [family] {table} {object}`

Stateful objects are attached to tables and are identified by an unique name. They group stateful information from rules, to reference them in rules the keywords "type name" are used e.g. "counter name".


`add`

<dd>

Add a new stateful object in the specified table.

</dd>

`delete`

<dd>

Delete the specified object.

</dd>

`list`

<dd>

Display stateful information the object holds.

</dd>

`reset`

<dd>

List-and-reset stateful object.

</dd>


### Ct

***ct*** {helper} {type} {type} {protocol} {protocol} [l3proto] [family]

Ct helper is used to define connection tracking helpers that can then be used in combination with the "ct helper set" statement. type and protocol are mandatory, l3proto is derived from the table family by default, i.e. in the inet table the kernel will try to load both the ipv4 and ipv6 helper backends, if they are supported by the kernel.


***Table 6. conntrack helper specifications***

| Keyword | Description | Type |
| --- | --- | --- |
| type | name of helper type | quoted string (e.g. "ftp") |
| protocol | layer 4 protocol of the helper | string (e.g. tcp) |
| l3proto | layer 3 protocol of the helper | address family (e.g. ip) |



***示例 2. defining and assigning ftp helper***

Unlike iptables, helper assignment needs to be performed after the conntrack lookup has completed, for example with the default 0 hook priority.

~~~
table inet myhelpers {
  ct helper ftp-standard {
     type "ftp" protocol tcp
  }
  chain prerouting {
      type filter hook prerouting priority 0;
      tcp dport 21 ct helper set "ftp-standard"
  }
}

~~~


### Counter

***counter*** [packets bytes]


***Table 7. Counter specifications***

| Keyword | Description | Type |
| --- | --- | --- |
| packets | initial count of packets | unsigned integer (64 bit) |
| bytes | initial count of bytes | unsigned integer (64 bit) |




### Quota

***quota*** [over | until] [used]


***Table 8. Quota specifications***

| Keyword | Description | Type |
| --- | --- | --- |
| quota | quota limit, used as the quota name | Two arguments, unsigned interger (64 bit) and string: bytes, kbytes, mbytes. "over" and "until" go before these arguments |
| used | initial value of used quota | Two arguments, unsigned interger (64 bit) and string: bytes, kbytes, mbytes |



## Expressions

Expressions represent values, either constants like network addresses, port numbers etc. or data gathered from the packet during ruleset evaluation. Expressions can be combined using binary, logical, relational and other types of expressions to form complex or relational (match) expressions. They are also used as arguments to certain types of operations, like NAT, packet marking etc.

Each expression has a data type, which determines the size, parsing and representation of symbolic values and type compatibility with other expressions.


### describe command

`describe {expression}`

The ***describe*** command shows information about the type of an expression and its data type.



***示例 3. The `describe` command***

~~~
$ nft describe tcp flags
payload expression, datatype tcp_flag (TCP flag) (basetype bitmask, integer), 8 bits

pre-defined symbolic constants:
fin                               0x01
syn                               0x02
rst                               0x04
psh                               0x08
ack                               0x10
urg                               0x20
ecn                               0x40
cwr                               0x80

~~~



## Data types

Data types determine the size, parsing and representation of symbolic values and type compatibility of expressions. A number of global data types exist, in addition some expression types define further data types specific to the expression type. Most data types have a fixed size, some however may have a dynamic size, f.i. the string type.

Types may be derived from lower order types, f.i. the IPv4 address type is derived from the integer type, meaning an IPv4 address can also be specified as an integer value.

In certain contexts (set and map definitions) it is necessary to explicitly specify a data type. Each type has a name which is used for this.


### Integer type


***Table 9.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Integer | integer | variable | - |


The integer type is used for numeric values. It may be specified as decimal, hexadecimal or octal number. The integer type doesn't have a fixed size, its size is determined by the expression for which it is used.


### Bitmask type


***Table 10.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Bitmask | bitmask | variable | integer |


The bitmask type (***bitmask***) is used for bitmasks.



### String type


***Table 11.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| String | string | variable | - |


The string type is used to for character strings. A string begins with an alphabetic character (a-zA-Z) followed by zero or more alphanumeric characters or the characters /, -, _ and .. In addition anything enclosed in double quotes (") is recognized as a string.


***示例 4. String specification***

~~~

# Interface name
filter input iifname eth0

# Weird interface name
filter input iifname "(eth0)"

~~~



### Link layer address type


***Table 12.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Link layer address | lladdr | variable | integer |


The link layer address type is used for link layer addresses. Link layer addresses are specified as a variable amount of groups of two hexadecimal digits separated using colons (:).


***示例 5. Link layer address specification***

~~~

# Ethernet destination MAC address
filter input ether daddr 20:c9:d0:43:12:d9

~~~



### IPv4 address type


***Table 13.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| IPv4 address | ipv4_addr | 32 bit | integer |


The IPv4 address type is used for IPv4 addresses. Addresses are specified in either dotted decimal, dotted hexadecimal, dotted octal, decimal, hexadecimal, octal notation or as a host name. A host name will be resolved using the standard system resolver.


***示例 6. IPv4 address specification***

~~~
# dotted decimal notation
filter output ip daddr 127.0.0.1

# host name
filter output ip daddr localhost

~~~



### IPv6 address type


***Table 14.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| IPv6 address | ipv6_addr | 128 bit | integer |


The IPv6 address type is used for IPv6 addresses. FIXME


***示例 7. IPv6 address specification***

~~~
# abbreviated loopback address
filter output ip6 daddr ::1

~~~



### Boolean type


***Table 15.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Boolean | boolean | 1 bit | integer |


The boolean type is a syntactical helper type in user space. It's use is in the right-hand side of a (typically implicit) relational expression to change the expression on the left-hand side into a boolean check (usually for existence).

The following keywords will automatically resolve into a boolean type with given value:


***Table 16.***

| Keyword | Value |
| --- | --- |
| exists | 1 |
| missing | 0 |


***示例 8. Boolean specification***

The following expressions support a boolean comparison:


***Table 17.***

| Expression | Behaviour |
| --- | --- |
| fib | Check route existence. |
| exthdr | Check IPv6 extension header existence. |
| tcp option | Check TCP option header existence. |


~~~
# match if route exists
filter input fib daddr . iif oif exists

# match only non-fragmented packets in IPv6 traffic
filter input exthdr frag missing

# match if TCP timestamp option is present
filter input tcp option timestamp exists

~~~




### ICMP Type type


***Table 18.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| ICMP Type | icmp_type | 8 bit | integer |


The ICMP Type type is used to conveniently specify the ICMP header's type field.

The following keywords may be used when specifying the ICMP type:


***Table 19.***

| Keyword | Value |
| --- | --- |
| echo-reply | 0 |
| destination-unreachable | 3 |
| source-quench | 4 |
| redirect | 5 |
| echo-request | 8 |
| router-advertisement | 9 |
| router-solicitation | 10 |
| time-exceeded | 11 |
| parameter-problem | 12 |
| timestamp-request | 13 |
| timestamp-reply | 14 |
| info-request | 15 |
| info-reply | 16 |
| address-mask-request | 17 |
| address-mask-reply | 18 |




***示例 9. ICMP Type specification***

~~~

# match ping packets
filter output icmp type { echo-request, echo-reply }

~~~






### ICMPv6 Type type


***Table 20.***

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| ICMPv6 Type | icmpv6_type | 8 bit | integer |



The ICMPv6 Type type is used to conveniently specify the ICMPv6 header's type field.

The following keywords may be used when specifying the ICMPv6 type:


***Table 21.***

| Keyword | Value |
| --- | --- |
| destination-unreachable | 1 |
| packet-too-big | 2 |
| time-exceeded | 3 |
| parameter-problem | 4 |
| echo-request | 128 |
| echo-reply | 129 |
| mld-listener-query | 130 |
| mld-listener-report | 131 |
| mld-listener-done | 132 |
| mld-listener-reduction | 132 |
| nd-router-solicit | 133 |
| nd-router-advert | 134 |
| nd-neighbor-solicit | 135 |
| nd-neighbor-advert | 136 |
| nd-redirect | 137 |
| router-renumbering | 138 |
| ind-neighbor-solicit | 141 |
| ind-neighbor-advert | 142 |
| mld2-listener-report | 143 |


***示例 10. ICMPv6 Type specification***

~~~

# match ICMPv6 ping packets
filter output icmpv6 type { echo-request, echo-reply }

~~~



## Primary expressions

The lowest order expression is a primary expression, representing either a constant or a single datum from a packet's payload, meta data or a stateful module.


### Meta expressions

`meta {length | nfproto | l4proto | protocol | priority}`

`[meta] {mark | iif | iifname | iiftype | oif | oifname | oiftype}`
`[meta] {skuid | skgid | nftrace | rtclassid | ibriport | obriport | pkttype | cpu | iifgroup | oifgroup | cgroup | random}`

A meta expression refers to meta data associated with a packet.

There are two types of meta expressions: unqualified and qualified meta expressions. Qualified meta expressions require the ***meta*** keyword before the meta key, unqualified meta expressions can be specified by using the meta key directly or as qualified meta expressions.


***Table 22. Meta expression types***

| Keyword | Description | Type |
| --- | --- | --- |
| length | Length of the packet in bytes | integer (32 bit) |
| protocol | Ethertype protocol value | ether_type |
| priority | TC packet priority | tc_handle |
| mark | Packet mark | mark |
| iif | Input interface index | iface_index |
| iifname | Input interface name | string |
| iiftype | Input interface type | iface_type |
| oif | Output interface index | iface_index |
| oifname | Output interface name | string |
| oiftype | Output interface hardware type | iface_type |
| skuid | UID associated with originating socket | uid |
| skgid | GID associated with originating socket | gid |
| rtclassid | Routing realm | realm |
| ibriport | Input bridge interface name | string |
| obriport | Output bridge interface name | string |
| pkttype | packet type | pkt_type |
| cpu | cpu number processing the packet | integer (32 bits) |
| iifgroup | incoming device group | devgroup |
| oifgroup | outgoing device group | devgroup |
| cgroup | control group id | integer (32 bits) |
| random | pseudo-random number | integer (32 bits) |




***Table 23. Meta expression specific types***

| Type | Description |
| --- | --- |
| iface_index | Interface index (32 bit number). Can be specified numerically or as name of an existing interface. |
| ifname | Interface name (16 byte string). Does not have to exist. |
| iface_type | Interface type (16 bit number). |
| uid | User ID (32 bit number). Can be specified numerically or as user name. |
| gid | Group ID (32 bit number). Can be specified numerically or as group name. |
| realm | Routing Realm (32 bit number). Can be specified numerically or as symbolic name defined in /etc/iproute2/rt_realms. |
| devgroup_type | Device group (32 bit number). Can be specified numerically or as symbolic name defined in /etc/iproute2/group. |
| pkt_type | Packet type: Unicast (addressed to local host), Broadcast (to all), Multicast (to group). |




***示例 11. Using meta expressions***

~~~

# qualified meta expression
filter output meta oif eth0

# unqualified meta expression
filter output oif eth0

~~~






### fib expressions

***fib*** {saddr | daddr [mark | iif | oif]} {oif | oifname | type}

A fib expression queries the fib (forwarding information base) to obtain information such as the output interface index a particular address would use. The input is a tuple of elements that is used as input to the fib lookup functions.


***Table 24. fib expression specific types***

| Keyword | Description | Type |
| --- | --- | --- |
| oif | Output interface index | integer (32 bit) |
| oifname | Output interface name | string |
| type | Address type | fib_addrtype |




***示例 12. Using fib expressions***

~~~

# drop packets without a reverse path
filter prerouting fib saddr . iif oif missing drop

# drop packets to address not configured on ininterface
filter prerouting fib daddr . iif type != { local, broadcast, multicast } drop

# perform lookup in a specific 'blackhole' table (0xdead, needs ip appropriate ip rule)
filter prerouting meta mark set 0xdead fib daddr . mark type vmap { blackhole : drop, prohibit : jump prohibited, unreachable : drop }

~~~






### Routing expressions

***rt*** {classid | nexthop}

A routing expression refers to routing data associated with a packet.


***Table 25. Routing expression types***

| Keyword | Description | Type |
| --- | --- | --- |
| classid | Routing realm | realm |
| nexthop | Routing nexthop | ipv4_addr/ipv6_addr |




***Table 26. Routing expression specific types***

| Type | Description |
| --- | --- |
| realm | Routing Realm (32 bit number). Can be specified numerically or as symbolic name defined in /etc/iproute2/rt_realms. |




***示例 13. Using routing expressions***

~~~

# IP family independent rt expression
filter output rt classid 10

# IP family dependent rt expressions
ip filter output rt nexthop 192.168.0.1
ip6 filter output rt nexthop fd00::1
inet filter meta nfproto ipv4 output rt nexthop 192.168.0.1
inet filter meta nfproto ipv6 output rt nexthop fd00::1

~~~








## Payload expressions

Payload expressions refer to data from the packet's payload.


### Ethernet header expression

***ether*** [_ethernet header field_]


***Table 27. Ethernet header expression types***

| Keyword | Description | Type |
| --- | --- | --- |
| daddr | Destination MAC address | ether_addr |
| saddr | Source MAC address | ether_addr |
| type | EtherType | ether_type |






### VLAN header expression

***vlan*** [_VLAN header field_]


***Table 28. VLAN header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| id | VLAN ID (VID) | integer (12 bit) |
| cfi | Canonical Format Indicator | integer (1 bit) |
| pcp | Priority code point | integer (3 bit) |
| type | EtherType | ether_type |



### ARP header expression

***arp*** [_ARP header field_]


***Table 29. ARP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| htype | ARP hardware type | integer (16 bit) |
| ptype | EtherType | ether_type |
| hlen | Hardware address len | integer (8 bit) |
| plen | Protocol address len | integer (8 bit) |
| operation | Operation | arp_op |






### IPv4 header expression

***ip*** [_IPv4 header field_]


***Table 30. IPv4 header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| version | IP header version (4) | integer (4 bit) |
| hdrlength | IP header length including options | integer (4 bit) FIXME scaling |
| dscp | Differentiated Services Code Point | dscp |
| ecn | Explicit Congestion Notification | ecn |
| length | Total packet length | integer (16 bit) |
| id | IP ID | integer (16 bit) |
| frag-off | Fragment offset | integer (16 bit) |
| ttl | Time to live | integer (8 bit) |
| protocol | Upper layer protocol | inet_proto |
| checksum | IP header checksum | integer (16 bit) |
| saddr | Source address | ipv4_addr |
| daddr | Destination address | ipv4_addr |






### ICMP header expression

***icmp*** [_ICMP header field_]


***Table 31. ICMP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| type | ICMP type field | icmp_type |
| code | ICMP code field | integer (8 bit) |
| checksum | ICMP checksum field | integer (16 bit) |
| id | ID of echo request/response | integer (16 bit) |
| sequence | sequence number of echo request/response | integer (16 bit) |
| gateway | gateway of redirects | integer (32 bit) |
| mtu | MTU of path MTU discovery | integer (16 bit) |






### IPv6 header expression

***ip6*** [_IPv6 header field_]


***Table 32. IPv6 header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| version | IP header version (6) | integer (4 bit) |
| dscp | Differentiated Services Code Point | dscp |
| ecn | Explicit Congestion Notification | ecn |
| flowlabel | Flow label | integer (20 bit) |
| length | Payload length | integer (16 bit) |
| nexthdr | Nexthdr protocol | inet_proto |
| hoplimit | Hop limit | integer (8 bit) |
| saddr | Source address | ipv6_addr |
| daddr | Destination address | ipv6_addr |






### ICMPv6 header expression

***icmpv6*** [_ICMPv6 header field_]


***Table 33. ICMPv6 header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| type | ICMPv6 type field | icmpv6_type |
| code | ICMPv6 code field | integer (8 bit) |
| checksum | ICMPv6 checksum field | integer (16 bit) |
| parameter-problem | pointer to problem | integer (32 bit) |
| packet-too-big | oversized MTU | integer (32 bit) |
| id | ID of echo request/response | integer (16 bit) |
| sequence | sequence number of echo request/response | integer (16 bit) |
| max-delay | maximum response delay of MLD queries | integer (16 bit) |






### TCP header expression

***tcp*** [_TCP header field_]


***Table 34. TCP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| sequence | Sequence number | integer (32 bit) |
| ackseq | Acknowledgement number | integer (32 bit) |
| doff | Data offset | integer (4 bit) FIXME scaling |
| reserved | Reserved area | integer (4 bit) |
| flags | TCP flags | tcp_flag |
| window | Window | integer (16 bit) |
| checksum | Checksum | integer (16 bit) |
| urgptr | Urgent pointer | integer (16 bit) |






### UDP header expression

***udp*** [_UDP header field_]


***Table 35. UDP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| length | Total packet length | integer (16 bit) |
| checksum | Checksum | integer (16 bit) |






### UDP-Lite header expression

***udplite*** [_UDP-Lite header field_]


***Table 36. UDP-Lite header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| checksum | Checksum | integer (16 bit) |






### SCTP header expression

***sctp*** [_SCTP header field_]


***Table 37. SCTP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| vtag | Verfication Tag | integer (32 bit) |
| checksum | Checksum | integer (32 bit) |






### DCCP header expression

***dccp*** [_DCCP header field_]


***Table 38. DCCP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |






### Authentication header expression

***ah*** [_AH header field_]


***Table 39. AH header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| nexthdr | Next header protocol | inet_proto |
| hdrlength | AH Header length | integer (8 bit) |
| reserved | Reserved area | integer (16 bit) |
| spi | Security Parameter Index | integer (32 bit) |
| sequence | Sequence number | integer (32 bit) |






### Encrypted security payload header expression

***esp*** [_ESP header field_]


***Table 40. ESP header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| spi | Security Parameter Index | integer (32 bit) |
| sequence | Sequence number | integer (32 bit) |






### IPcomp header expression

***comp*** [_IPComp header field_]


***Table 41. IPComp header expression***

| Keyword | Description | Type |
| --- | --- | --- |
| nexthdr | Next header protocol | inet_proto |
| flags | Flags | bitmask |
| cpi | Compression Parameter Index | integer (16 bit) |






### Extension header expressions

Extension header expressions refer to data from variable-sized protocol headers, such as IPv6 extension headers and TCPs options.

nftables currently supports matching (finding) a given ipv6 extension header or TCP option.

`hbh {nexthdr | hdrlength}`

`frag {nexthdr | frag-off | more-fragments | id}`

`rt {nexthdr | hdrlength | type | seg-left}`

`dst {nexthdr | hdrlength}`

`mh {nexthdr | hdrlength | checksum | type}`

`tcp option {eol | noop | maxseg | window | sack-permitted | sack | sack0 | sack1 | sack2 | sack3 | timestamp} [_tcp_option_field_]`

The following syntaxes are valid only in a relational expression with boolean type on right-hand side for checking header existence only:

`exthdr {hbh | frag | rt | dst | mh}`

`tcp option {eol | noop | maxseg | window | sack-permitted | sack | sack0 | sack1 | sack2 | sack3 | timestamp}`


***Table 42. IPv6 extension headers***

| Keyword | Description |
| --- | --- |
| hbh | Hop by Hop |
| rt | Routing Header |
| frag | Fragmentation header |
| dst | dst options |
| mh | Mobility Header |




***Table 43. TCP Options***

| Keyword | Description | TCP option fields |
| --- | --- | --- |
| eol | End of option list | kind |
| noop | 1 Byte TCP No-op options | kind |
| maxseg | TCP Maximum Segment Size | kind, length, size |
| window | TCP Window Scaling | kind, length, count |
| sack-permitted | TCP SACK permitted | kind, length |
| sack | TCP Selective Acknowledgement (alias of block 0) | kind, length, left, right |
| sack0 | TCP Selective Acknowledgement (block 0) | kind, length, left, right |
| sack1 | TCP Selective Acknowledgement (block 1) | kind, length, left, right |
| sack2 | TCP Selective Acknowledgement (block 2) | kind, length, left, right |
| sack3 | TCP Selective Acknowledgement (block 3) | kind, length, left, right |
| timestamp | TCP Timestamps | kind, length, tsval, tsecr |




***示例 14. finding TCP options***

~~~
filter input tcp option sack-permitted kind 1 counter
~~~




***示例 15. matching IPv6 exthdr***

~~~
ip6 filter input frag more-fragments 1 counter
~~~



### Conntrack expressions

Conntrack expressions refer to meta data of the connection tracking entry associated with a packet.

There are three types of conntrack expressions. Some conntrack expressions require the flow direction before the conntrack key, others must be used directly because they are direction agnostic. The ***packets***, ***bytes*** and ***avgpkt*** keywords can be used with or without a direction. If the direction is omitted, the sum of the original and the reply direction is returned. The same is true for the ***zone***, if a direction is given, the zone is only matched if the zone id is tied to the given direction.

`ct {state | direction | status | mark | expiration | helper | label | l3proto | protocol | bytes | packets | avgpkt | zone}`

`ct {original | reply} {l3proto | protocol | saddr | daddr | proto-src | proto-dst | bytes | packets | avgpkt | zone}`


***Table 44. Conntrack expressions***

| Keyword | Description | Type |
| --- | --- | --- |
| state | State of the connection | ct_state |
| direction | Direction of the packet relative to the connection | ct_dir |
| status | Status of the connection | ct_status |
| mark | Connection mark | mark |
| expiration | Connection expiration time | time |
| helper | Helper associated with the connection | string |
| label | Connection tracking label bit or symbolic name defined in connlabel.conf in the nftables include path | ct_label |
| l3proto | Layer 3 protocol of the connection | nf_proto |
| saddr | Source address of the connection for the given direction | ipv4_addr/ipv6_addr |
| daddr | Destination address of the connection for the given direction | ipv4_addr/ipv6_addr |
| protocol | Layer 4 protocol of the connection for the given direction | inet_proto |
| proto-src | Layer 4 protocol source for the given direction | integer (16 bit) |
| proto-dst | Layer 4 protocol destination for the given direction | integer (16 bit) |
| packets | packet count seen in the given direction or sum of original and reply | integer (64 bit) |
| bytes | bytecount seen, see description for ***packets*** keyword | integer (64 bit) |
| avgpkt | average bytes per packet, see description for ***packets*** keyword | integer (64 bit) |
| zone | conntrack zone | integer (16 bit) |




## Statements

Statements represent actions to be performed. They can alter control flow (return, jump to a different chain, accept or drop the packet) or can perform actions, such as logging, rejecting a packet, etc.

Statements exist in two kinds. Terminal statements unconditionally terminate evaluation of the current rule, non-terminal statements either only conditionally or never terminate evaluation of the current rule, in other words, they are passive from the ruleset evaluation perspective. There can be an arbitrary amount of non-terminal statements in a rule, but only a single terminal statement as the final statement.


### Verdict statement

The verdict statement alters control flow in the ruleset and issues policy decisions for packets.

`{accept | drop | queue | continue | return}`

`{jump | goto} {chain}`


`accept`

<dd>

Terminate ruleset evaluation and accept the packet.

</dd>

`drop`

<dd>

Terminate ruleset evaluation and drop the packet.

</dd>

`queue`

<dd>

Terminate ruleset evaluation and queue the packet to userspace.

</dd>

`continue`

<dd>

Continue ruleset evaluation with the next rule. FIXME

</dd>

`return`

<dd>

Return from the current chain and continue evaluation at the next rule in the last chain. If issued in a base chain, it is equivalent to ***accept***.

</dd>

`jump chain`

<dd>

Continue evaluation at the first rule in chain. The current position in the ruleset is pushed to a call stack and evaluation will continue there when the new chain is entirely evaluated of a ***return*** verdict is issued.

</dd>

`goto chain`

<dd>

Similar to ***jump***, but the current position is not pushed to the call stack, meaning that after the new chain evaluation will continue at the last chain instead of the one containing the goto statement.

</dd>




***示例 16. Verdict statements***

~~~
# process packets from eth0 and the internal network in from_lan
# chain, drop all packets from eth0 with different source addresses.

filter input iif eth0 ip saddr 192.168.0.0/24 jump from_lan
filter input iif eth0 drop
~~~



### Payload statement

The payload statement alters packet content. It can be used for example to set ip DSCP (differv) header field or ipv6 flow labels.


***示例 17. route some packets instead of bridging***

~~~
# redirect tcp:http from 192.160.0.0/16 to local machine for routing instead of bridging
# assumes 00:11:22:33:44:55 is local MAC address.
bridge input meta iif eth0 ip saddr 192.168.0.0/16 tcp dport 80 meta pkttype set unicast ether daddr set 00:11:22:33:44:55
~~~


***示例 18. Set IPv4 DSCP header field***

~~~
ip forward ip dscp set 42
~~~



### Log statement

`log [prefix _quoted_string_] [level _syslog-level_] [flags log-flags]`

`log [group _nflog_group_] [prefix _quoted_string_] [queue-threshold value] [snaplen size]`

The log statement enables logging of matching packets. When this statement is used from a rule, the Linux kernel will print some information on all matching packets, such as header fields, via the kernel log (where it can be read with dmesg(1) or read in the syslog). If the group number is specified, the Linux kernel will pass the packet to nfnetlink_log which will multicast the packet through a netlink socket to the specified multicast group. One or more userspace processes may subscribe to the group to receive the packets, see libnetfilter_queue documentation for details. This is a non-terminating statement, so the rule evaluation continues after the packet is logged.


***Table 45. log statement options***

| Keyword | Description | Type |
| --- | --- | --- |
| prefix | Log message prefix | quoted string |
| syslog-level | Syslog level of logging | string: emerg, alert, crit, err, warn [default], notice, info, debug |
| group | NFLOG group to send messages to | unsigned integer (16 bit) |
| snaplen | Length of packet payload to include in netlink message | unsigned integer (32 bit) |
| queue-threshold | Number of packets to queue inside the kernel before sending them to userspace | unsigned integer (32 bit) |




***Table 46. log-flags***

| Flag | Description |
| --- | --- |
| tcp sequence | Log TCP sequence numbers. |
| tcp options | Log options from the TCP packet header. |
| ip options | Log options from the IP/IPv6 packet header. |
| skuid | Log the userid of the process which generated the packet. |
| ether | Decode MAC addresses and protocol. |
| all | Enable all log flags listed above. |




***示例 19. Using log statement***

~~~
# log the UID which generated the packet and ip options
ip filter output log flags skuid flags ip options

# log the tcp sequence numbers and tcp options from the TCP packet
ip filter output log flags tcp sequence,options

# enable all supported log flags
ip6 filter output log flags all
~~~


### Reject statement

`reject [with] {icmp | icmp6 | icmpx} [type] {icmp_type | icmp6_type | icmpx_type}`

`reject [with] {tcp} {reset}`

A reject statement is used to send back an error packet in response to the matched packet otherwise it is equivalent to drop so it is a terminating statement, ending rule traversal. This statement is only valid in the input, forward and output chains, and user-defined chains which are only called from those chains.


***Table 47. reject statement type (ip)***

| Value | Description | Type |
| --- | --- | --- |
| icmp_type | ICMP type response to be sent to the host | net-unreachable, host-unreachable, prot-unreachable, port-unreachable [default], net-prohibited, host-prohibited, admin-prohibited |




***Table 48. reject statement type (ip6)***

| Value | Description | Type |
| --- | --- | --- |
| icmp6_type | ICMPv6 type response to be sent to the host | no-route, admin-prohibited, addr-unreachable, port-unreachable [default], policy-fail, reject-route |


***Table 49. reject statement type (inet)***

| Value | Description | Type |
| --- | --- | --- |
| icmpx_type | ICMPvXtype abstraction response to be sent to the host, this is a set of types that overlap in IPv4 and IPv6 to be used from the inet family. | port-unreachable [default], admin-prohibited, no-route, host-unreachable |



### Counter statement

A counter statement sets the hit count of packets along with the number of bytes.

`counter {packets _number_ } {bytes _number_ }`



### Conntrack statement

The conntrack statement can be used to set the conntrack mark and conntrack labels.

`ct {mark | eventmask | label | zone} [set] value`

The ct statement sets meta data associated with a connection. The zone id has to be assigned before a conntrack lookup takes place, i.e. this has to be done in prerouting and possibly output (if locally generated packets need to be placed in a distinct zone), with a hook priority of -300.


***Table 50. Conntrack statement types***

| Keyword | Description | Value |
| --- | --- | --- |
| eventmask | conntrack event bits | bitmask, integer (32 bit) |
| helper | name of ct helper object to assign to the connection | quoted string |
| mark | Connection tracking mark | mark |
| label | Connection tracking label | label |
| zone | conntrack zone | integer (16 bit) |




***示例 20. save packet nfmark in conntrack***

~~~
ct mark set meta mark
~~~




***示例 21. set zone mapped via interface***

~~~
table inet raw {
  chain prerouting {
      type filter hook prerouting priority -300;
      ct zone set iif map { "eth1" : 1, "veth1" : 2 }
  }
  chain output {
      type filter hook output priority -300;
      ct zone set oif map { "eth1" : 1, "veth1" : 2 }
  }
}
~~~




***示例 22. restrict events reported by ctnetlink***

~~~
ct eventmask set new or related or destroy
~~~



### Meta statement

A meta statement sets the value of a meta expression. The existing meta fields are: priority, mark, pkttype, nftrace.

`meta {mark | priority | pkttype | nftrace} [set] value`

A meta statement sets meta data associated with a packet.


***Table 51. Meta statement types***

| Keyword | Description | Value |
| --- | --- | --- |
| priority | TC packet priority | tc_handle |
| mark | Packet mark | mark |
| pkttype | packet type | pkt_type |
| nftrace | ruleset packet tracing on/off. Use ***monitor trace*** command to watch traces | 0, 1 |



### Limit statement

`limit [rate] [over]_packet_number_ [/] {second | minute | hour | day} [burst _packet_number_ packets]`

`limit [rate] [over]_byte_number_ {bytes | kbytes | mbytes} [/] {second | minute | hour | day | week} [burst _byte_number_ bytes]`

A limit statement matches at a limited rate using a token bucket filter. A rule using this statement will match until this limit is reached. It can be used in combination with the log statement to give limited logging. The ***over*** keyword, that is optional, makes it match over the specified rate.


***Table 52. limit statement values***

| Value | Description | Type |
| --- | --- | --- |
| packet_number | Number of packets | unsigned integer (32 bit) |
| byte_number | Number of bytes | unsigned integer (32 bit) |



### NAT statements

`snat [to _address_ [:port]] [persistent, random, fully-random]`

`snat [to _address_ - _address_ [:_port_ - _port_]] [persistent, random, fully-random]`

`dnat [to _address_ [:_port_]] [persistent, random, fully-random]`

`dnat [to _address_ [:_port_ - _port_]] [persistent, random, fully-random]`

`masquerade [to [:_port_]] [persistent, random, fully-random]`

`masquerade [to [:_port_ - _port_]] [persistent, random, fully-random]`

`redirect [to [:_port_]] [persistent, random, fully-random]`

`redirect [to [:_port_ - _port_]] [persistent, random, fully-random]`

The nat statements are only valid from nat chain types.

The ***snat*** and ***masquerade*** statements specify that the source address of the packet should be modified. While ***snat*** is only valid in the postrouting and input chains, ***masquerade*** makes sense only in postrouting. The ***dnat*** and ***redirect*** statements are only valid in the prerouting and output chains, they specify that the destination address of the packet should be modified. You can use non-base chains which are called from base chains of nat chain type too. All future packets in this connection will also be mangled, and rules should cease being examined.

The ***masquerade*** statement is a special form of ***snat*** which always uses the outgoing interface's IP address to translate to. It is particularly useful on gateways with dynamic (public) IP addresses.

The ***redirect*** statement is a special form of ***dnat*** which always translates the destination address to the local host's one. It comes in handy if one only wants to alter the destination port of incoming traffic on different interfaces.

Note that all nat statements require both prerouting and postrouting base chains to be present since otherwise packets on the return path won't be seen by netfilter and therefore no reverse translation will take place.


***Table 53. NAT statement values***

| Expression | Description | Type |
| --- | --- | --- |
| address | Specifies that the source/destination address of the packet should be modified. You may specify a mapping to relate a list of tuples composed of arbitrary expression key with address value. | ipv4_addr, ipv6_addr, eg. abcd::1234, or you can use a mapping, eg. meta mark map { 10 : 192.168.1.2, 20 : 192.168.1.3 } |
| port | Specifies that the source/destination address of the packet should be modified. | port number (16 bits) |




***Table 54. NAT statement flags***

| Flag | Description |
| --- | --- |
| persistent | Gives a client the same source-/destination-address for each connection. |
| random | If used then port mapping will be randomized using a random seeded MD5 hash mix using source and destination address and destination port. |
| fully-random | If used then port mapping is generated based on a 32-bit pseudo-random algorithm. |




***示例 23. Using NAT statements***

~~~
# create a suitable table/chain setup for all further examples
add table nat
add chain nat prerouting { type nat hook prerouting priority 0; }
add chain nat postrouting { type nat hook postrouting priority 100; }

# translate source addresses of all packets leaving via eth0 to address 1.2.3.4
add rule nat postrouting oif eth0 snat to 1.2.3.4

# redirect all traffic entering via eth0 to destination address 192.168.1.120
add rule nat prerouting iif eth0 dnat to 192.168.1.120

# translate source addresses of all packets leaving via eth0 to whatever
# locally generated packets would use as source to reach the same destination
add rule nat postrouting oif eth0 masquerade

# redirect incoming TCP traffic for port 22 to port 2222
add rule nat prerouting tcp dport 22 redirect to :2222
~~~


### Queue statement

This statement passes the packet to userspace using the nfnetlink_queue handler. The packet is put into the queue identified by its 16-bit queue number. Userspace can inspect and modify the packet if desired. Userspace must then drop or reinject the packet into the kernel. See libnetfilter_queue documentation for details.

`queue [num _queue_number_] [bypass]`

`queue [num _queue_number_from_ - _queue_number_to_] [bypass,fanout]`


***Table 55. queue statement values***

| Value | Description | Type |
| --- | --- | --- |
| queue_number | Sets queue number, default is 0. | unsigned integer (16 bit) |
| queue_number_from | Sets initial queue in the range, if fanout is used. | unsigned integer (16 bit) |
| queue_number_to | Sets closing queue in the range, if fanout is used. | unsigned integer (16 bit) |



***Table 56. queue statement flags***

| Flag | Description |
| --- | --- |
| bypass | Let packets go through if userspace application cannot back off. Before using this flag, read libnetfilter_queue documentation for performance tuning recomendations. |
| fanout | Distribute packets between several queues. |



## Additional commands

These are some additional commands included in nft.


### export

Export your current ruleset in XML or JSON format to stdout.

Examples:

~~~
[...]
% nft export json
[...]
~~~



### monitor

The monitor command allows you to listen to Netlink events produced by the nf_tables subsystem, related to creation and deletion of objects. When they ocurr, nft will print to stdout the monitored events in either XML, JSON or native nft format.

To filter events related to a concrete object, use one of the keywords 'tables', 'chains', 'sets', 'rules', 'elements'.

To filter events related to a concrete action, use keyword 'new' or 'destroy'.

Hit ^C to finish the monitor operation.


***示例 24. Listen to all events, report in native nft format***

~~~
% nft monitor
~~~



***示例 25. Listen to added tables, report in XML format***

~~~
% nft monitor new tables xml
~~~




***示例 26. Listen to deleted rules, report in JSON format***

~~~
% nft monitor destroy rules json
~~~




***示例 27. Listen to both new and destroyed chains, in native nft format***

~~~
% nft monitor chains
~~~


## Error reporting

When an error is detected, nft shows the line(s) containing the error, the position of the erroneous parts in the input stream and marks up the erroneous parts using carrets (^). If the error results from the combination of two expressions or statements, the part imposing the constraints which are violated is marked using tildes (~).

For errors returned by the kernel, nft can't detect which parts of the input caused the error and the entire command is marked.


***示例 28. Error caused by single incorrect expression***

~~~
<cmdline>:1:19-22: Error: Interface does not exist
filter output oif eth0
                  ^^^^
~~~




***示例 29. Error caused by invalid combination of two expressions***

~~~
<cmdline>:1:28-36: Error: Right hand side of relational expression (==) must be constant
filter output tcp dport == tcp dport
                        ~~ ^^^^^^^^^
~~~




***示例 30. Error returned by the kernel***

~~~
<cmdline>:0:0-23: Error: Could not process rule: Operation not permitted
filter output oif wlan0
^^^^^^^^^^^^^^^^^^^^^^^
~~~


## 退出状态码

On success, nft exits with a status of 0. Unspecified errors cause it to exit with a status of 1, memory allocation errors with a status of 2, unable to open Netlink socket with 3.



## See Also

iptables(8), ip6tables(8), arptables(8), ebtables(8), ip(8), tc(8)

There is an official wiki at: [wiki.nftables.org](http://wiki.nftables.org)



## Authors

nftables was written by Patrick McHardy and Pablo Neira Ayuso, among many other contributors from the Netfilter community.



## Copyright

* Copyright © 2008-2014 Patrick McHardy <[kaber@trash.net](mailto:kaber@trash.net)>
* Copyright © 2013-2016 Pablo Neira Ayuso <[pablo@netfilter.org](mailto:pablo@netfilter.org)>

nftables is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License version 2 as published by the Free Software Foundation.

***This documentation is licenced under the terms of the Creative Commons Attribution-ShareAlike 4.0 license, [CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/).***

