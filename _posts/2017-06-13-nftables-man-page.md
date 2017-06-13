---
layout: post
title: "nftables：nft man文档"
description: "这里是一份nft man文档"
categories: [system]
tags: [nft, nftables ]
---

> 最近利用空闲时间学习了nftables的基础知识，将man文档翻译一遍加深记忆。

* Kramdown table of contents
{:toc .toc}


## Name

nft --  Administration tool for packet filtering and classification


## Synopsis

**nft** [ `-n | --numeric` ] [ `-s | --stateless` ] [ `[-I | --includepath]` <tt class="replaceable">_directory_</tt> ] [ `[-f | --file]` <tt class="replaceable">_filename_</tt> | `[-i | --interactive]` | <tt class="replaceable">_cmd_</tt> ]

**nft** [ `-h | --help` ] [ `-v | --version` ]



## Description

nft is used to set up, maintain and inspect packet filtering and classification rules in the Linux kernel.

</div>


## Options

For a full summary of options, run **nft --help**.

<div class="variablelist">

<dl>

<dt>`-h, --help`</dt>

<dd>

Show help message and all options.

</dd>

<dt>`-v, --version`</dt>

<dd>

Show version.

</dd>

<dt>`-n, --numeric`</dt>

<dd>

Show data numerically. When used once (the default behaviour), skip lookup of addresses to symbolic names. Use twice to also show Internet services (port numbers) numerically. Use three times to also show protocols and UIDs/GIDs numerically.

</dd>

<dt>`-s, --stateless`</dt>

<dd>

Omit stateful information of rules and stateful objects.

</dd>

<dt>`-N`</dt>

<dd>

Translate IP addresses to names. Usually requires network traffic for DNS lookup.

</dd>

<dt>`-a, --handle`</dt>

<dd>

Show rule handles in output.

</dd>

<dt>`-I, --includepath <tt class="replaceable">_directory_</tt>`</dt>

<dd>

Add the directory <tt class="replaceable">_directory_</tt> to the list of directories to be searched for included files.

</dd>

<dt>`-f, --file <tt class="replaceable">_filename_</tt>`</dt>

<dd>

Read input from <tt class="replaceable">_filename_</tt>.

</dd>

<dt>`-i, --interactive`</dt>

<dd>

Read input from an interactive readline CLI.

</dd>

</dl>

</div>

</div>


## Input file format

<div class="refsect2"><a name="AEN107"></a>

### Lexical conventions

Input is parsed line-wise. When the last character of a line, just before the newline character, is a non-quoted backslash (<tt class="literal">\</tt>), the next line is treated as a continuation. Multiple commands on the same line can be separated using a semicolon (<tt class="literal">;</tt>).

A hash sign (<tt class="literal">#</tt>) begins a comment. All following characters on the same line are ignored.

Identifiers begin with an alphabetic character (<tt class="literal">a-z,A-Z</tt>), followed zero or more alphanumeric characters (<tt class="literal">a-z,A-Z,0-9</tt>) and the characters slash (<tt class="literal">/</tt>), backslash (<tt class="literal">\</tt>), underscore (<tt class="literal">_</tt>) and dot (<tt class="literal">.</tt>). Identifiers using different characters or clashing with a keyword need to be enclosed in double quotes (<tt class="literal">"</tt>).

</div>

<div class="refsect2"><a name="AEN123"></a>

### Include files

**include**"<tt class="replaceable">_filename_</tt>"

Other files can be included by using the **include** statement. The directories to be searched for include files can be specified using the `-I/--includepath` option.

If the <tt class="literal">filename</tt> parameter is a directory, then all files in the directory are loaded in alphabetical order.

</div>

<div class="refsect2"><a name="AEN134"></a>

### Symbolic variables

**define****`<tt class="replaceable">_variable_</tt>` = <tt class="replaceable">_expr_</tt>**

**<div class="refsect2"<tt class="replaceable">_variable_</tt>`**

Symbolic variables can be defined using the **define** statement. Variable references are expressions and can be used initialize other variables. The scope of a definition is the current block and all blocks contained within.

<div class="example"><a name="AEN149"></a>

**Example 1\. Using symbolic variables**

| 

<pre class="programlisting">define int_if1 = eth0
define int_if2 = eth1
define int_ifs = { $int_if1, $int_if2 }

filter input iif $int_ifs accept
                    </pre>

 |

</div>

</div>

</div>


## Address families

Address families determine the type of packets which are processed. For each address family the kernel contains so called hooks at specific stages of the packet processing paths, which invoke nftables if rules for these hooks exist.

<div class="variablelist">

<dl>

<dt>`ip`</dt>

<dd>

IPv4 address family.

</dd>

<dt>`ip6`</dt>

<dd>

IPv6 address family.

</dd>

<dt>`inet`</dt>

<dd>

Internet (IPv4/IPv6) address family.

</dd>

<dt>`arp`</dt>

<dd>

ARP address family, handling packets vi

</dd>

<dt>`bridge`</dt>

<dd>

Bridge address family, handling packets which traverse a bridge device.

</dd>

<dt>`netdev`</dt>

<dd>

Netdev address family, handling packets from ingress.

</dd>

</dl>

</div>

All nftables objects exist in address family specific namespaces, therefore all identifiers include an address family. If an identifier is specified without an address family, the <tt class="literal">ip</tt> family is used by default.

<div class="refsect2"><a name="AEN189"></a>

### IPv4/IPv6/Inet address families

The IPv4/IPv6/Inet address families handle IPv4, IPv6 or both types of packets. They contain five hooks at different packet processing stages in the network stack.

<div class="table"><a name="AEN193"></a>

**Table 1\. IPv4/IPv6/Inet address family hooks**

| Hook | Description |
| --- | --- |
| prerouting | All packets entering the system are processed by the prerouting hook. It is invoked before the routing process and is used for early filtering or changing packet attributes that affect routing. |
| input | Packets delivered to the local system are processed by the input hook. |
| forward | Packets forwarded to a different host are processed by the forward hook. |
| output | Packets sent by local processes are processed by the output hook. |
| postrouting | All packets leaving the system are processed by the postrouting hook. |

</div>

</div>

<div class="refsect2"><a name="AEN218"></a>

### ARP address family

The ARP address family handles ARP packets received and sent by the system. It is commonly used to mangle ARP packets for clustering.

<div class="table"><a name="AEN222"></a>

**Table 2\. ARP address family hooks**

| Hook | Description |
| --- | --- |
| input | Packets delivered to the local system are processed by the input hook. |
| output | Packets send by the local system are processed by the output hook. |

</div>

</div>

<div class="refsect2"><a name="AEN238"></a>

### Bridge address family

The bridge address family handles ethernet packets traversing bridge devices.

</div>

<div class="refsect2"><a name="AEN241"></a>

### Netdev address family

The Netdev address family handles packets from ingress.

<div class="table"><a name="AEN245"></a>

**Table 3\. Netdev address family hooks**

| Hook | Description |
| --- | --- |
| ingress | All packets entering the system are processed by this hook. It is invoked before layer 3 protocol handlers and it can be used for early filtering and policing. |

</div>

</div>

</div>


## Tables

{add | delete | list | flush}**table** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>}

Tables are containers for chains, sets and stateful objects. They are identified by their address family and their name. The address family must be one of <tt class="literal">ip</tt>, <tt class="literal">ip6</tt>, <tt class="literal">inet</tt>, <tt class="literal">arp</tt>, <tt class="literal">bridge</tt>, <tt class="literal">netdev</tt>. The <tt class="literal">inet</tt> address family is a dummy family which is used to create hybrid IPv4/IPv6 tables. When no address family is specified, <tt class="literal">ip</tt> is used by default.

<div class="variablelist">

<dl>

<dt>`add`</dt>

<dd>

Add a new table for the given family with the given name.

</dd>

<dt>`delete`</dt>

<dd>

Delete the specified table.

</dd>

<dt>`list`</dt>

<dd>

List all chains and rules of the specified table.

</dd>

<dt>`flush`</dt>

<dd>

Flush all chains and rules of the specified table.

</dd>

</dl>

</div>

</div>


## Chains

{add}**chain** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_chain_</tt>} {<tt class="replaceable">_hook_</tt>} {<tt class="replaceable">_priority_</tt>} {<tt class="replaceable">_policy_</tt>} {<tt class="replaceable">_device_</tt>}

{add | create | delete | list | flush}**chain** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_chain_</tt>}

{rename}**chain** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_chain_</tt>} {<tt class="replaceable">_newname_</tt>}

Chains are containers for rules. They exist in two kinds, base chains and regular chains. A base chain is an entry point for packets from the networking stack, a regular chain may be used as jump target and is used for better rule organization.

<div class="variablelist">

<dl>

<dt>`add`</dt>

<dd>

Add a new chain in the specified table. When a hook and priority value are specified, the chain is created as a base chain and hooked up to the networking stack.

</dd>

<dt>`create`</dt>

<dd>

Similar to the **add** command, but returns an error if the chain already exists.

</dd>

<dt>`delete`</dt>

<dd>

Delete the specified chain. The chain must not contain any rules or be used as jump target.

</dd>

<dt>`rename`</dt>

<dd>

Rename the specified chain.

</dd>

<dt>`list`</dt>

<dd>

List all rules of the specified chain.

</dd>

<dt>`flush`</dt>

<dd>

Flush all rules of the specified chain.

</dd>

</dl>

</div>

</div>


## Rules

[add | insert]**rule** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_chain_</tt>} [position <tt class="replaceable">_position_</tt>] {<tt class="replaceable">_statement_</tt>...}

{delete}**rule** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_chain_</tt>} {handle <tt class="replaceable">_handle_</tt>}

Rules are constructed from two kinds of components according to a set of grammatical rules: expressions and statements.

<div class="variablelist">

<dl>

<dt>`add`</dt>

<dd>

Add a new rule described by the list of statements. The rule is appended to the given chain unless a position is specified, in which case the rule is appended to the rule given by the position.

</dd>

<dt>`insert`</dt>

<dd>

Similar to the **add** command, but the rule is prepended to the beginning of the chain or before the rule at the given position.

</dd>

<dt>`delete`</dt>

<dd>

Delete the specified rule.

</dd>

</dl>

</div>

</div>


## Sets

{add} **set** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_set_</tt>}{ {<tt class="replaceable">_type_</tt>} [<tt class="replaceable">_flags_</tt>] [<tt class="replaceable">_timeout_</tt>] [<tt class="replaceable">_gc-interval_</tt>] [<tt class="replaceable">_elements_</tt>] [<tt class="replaceable">_size_</tt>] [<tt class="replaceable">_policy_</tt>]}

{delete | list | flush} **set** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_set_</tt>}

{add | delete} **element** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_set_</tt>}{ {<tt class="replaceable">_elements_</tt>}}

Sets are elements containers of an user-defined data type, they are uniquely identified by an user-defined name and attached to tables.

<div class="variablelist">

<dl>

<dt>`add`</dt>

<dd>

Add a new set in the specified table.

</dd>

<dt>`delete`</dt>

<dd>

Delete the specified set.

</dd>

<dt>`list`</dt>

<dd>

Display the elements in the specified set.

</dd>

<dt>`flush`</dt>

<dd>

Remove all elements from the specified set.

</dd>

<dt>`add element`</dt>

<dd>

Comma-separated list of elements to add into the specified set.

</dd>

<dt>`delete element`</dt>

<dd>

Comma-separated list of elements to delete from the specified set.

</dd>

</dl>

</div>

<div class="table"><a name="AEN517"></a>

**Table 4\. Set specifications**

| Keyword | Description | Type |
| --- | --- | --- |
| type | data type of set elements | string: ipv4_addr, ipv6_addr, ether_addr, inet_proto, inet_service, mark |
| flags | set flags | string: constant, interval, timeout |
| timeout | time an element stays in the set | string, decimal followed by unit. Units are: d, h, m, s |
| gc-interval | garbage collection interval, only available when timeout or flag timeout are active | string, decimal followed by unit. Units are: d, h, m, s |
| elements | elements contained by the set | set data type |
| size | maximun number of elements in the set | unsigned integer (64 bit) |
| policy | set policy | string: performance [default], memory |

</div>

</div>


## Maps

{add} **map** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_map_</tt>}{ {<tt class="replaceable">_type_</tt>} [<tt class="replaceable">_flags_</tt>] [<tt class="replaceable">_elements_</tt>] [<tt class="replaceable">_size_</tt>] [<tt class="replaceable">_policy_</tt>]}

{delete | list | flush} **map** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_map_</tt>}

{add | delete} **element** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_map_</tt>}{ {<tt class="replaceable">_elements_</tt>}}

Maps store data based on some specific key used as input, they are uniquely identified by an user-defined name and attached to tables.

<div class="variablelist">

<dl>

<dt>`add`</dt>

<dd>

Add a new map in the specified table.

</dd>

<dt>`delete`</dt>

<dd>

Delete the specified map.

</dd>

<dt>`list`</dt>

<dd>

Display the elements in the specified map.

</dd>

<dt>`flush`</dt>

<dd>

Remove all elements from the specified map.

</dd>

<dt>`add element`</dt>

<dd>

Comma-separated list of elements to add into the specified map.

</dd>

<dt>`delete element`</dt>

<dd>

Comma-separated list of element keys to delete from the specified map.

</dd>

</dl>

</div>

<div class="table"><a name="AEN636"></a>

**Table 5\. Map specifications**

| Keyword | Description | Type |
| --- | --- | --- |
| type | data type of map elements | string ':' string: ipv4_addr, ipv6_addr, ether_addr, inet_proto, inet_service, mark, counter, quota. Counter and quota can't be used as keys |
| flags | map flags | string: constant, interval |
| elements | elements contained by the map | map data type |
| size | maximun number of elements in the map | unsigned integer (64 bit) |
| policy | map policy | string: performance [default], memory |

</div>

</div>


## Stateful objects

{add | delete | list | reset} **type** [<tt class="replaceable">_family_</tt>] {<tt class="replaceable">_table_</tt>} {<tt class="replaceable">_object_</tt>}

Stateful objects are attached to tables and are identified by an unique name. They group stateful information from rules, to reference them in rules the keywords "type name" are used e.g. "counter name".

<div class="variablelist">

<dl>

<dt>`add`</dt>

<dd>

Add a new stateful object in the specified table.

</dd>

<dt>`delete`</dt>

<dd>

Delete the specified object.

</dd>

<dt>`list`</dt>

<dd>

Display stateful information the object holds.

</dd>

<dt>`reset`</dt>

<dd>

List-and-reset stateful object.

</dd>

</dl>

</div>

<div class="refsect2"><a name="AEN706"></a>

### Ct

**ct** {helper} {type} {<tt class="replaceable">_type_</tt>} {protocol} {<tt class="replaceable">_protocol_</tt>} [l3proto] [<tt class="replaceable">_family_</tt>]

Ct helper is used to define connection tracking helpers that can then be used in combination with the <tt class="literal">"ct helper set"</tt> statement. type and protocol are mandatory, l3proto is derived from the table family by default, i.e. in the inet table the kernel will try to load both the ipv4 and ipv6 helper backends, if they are supported by the kernel.

<div class="table"><a name="AEN723"></a>

**Table 6\. conntrack helper specifications**

| Keyword | Description | Type |
| --- | --- | --- |
| type | name of helper type | quoted string (e.g. "ftp") |
| protocol | layer 4 protocol of the helper | string (e.g. tcp) |
| l3proto | layer 3 protocol of the helper | address family (e.g. ip) |

</div>

<div class="example"><a name="AEN747"></a>

**Example 2\. defining and assigning ftp helper**

Unlike iptables, helper assignment needs to be performed after the conntrack lookup has completed, for example with the default 0 hook priority.

| 

<pre class="programlisting">table inet myhelpers {
  ct helper ftp-standard {
     type "ftp" protocol tcp
  }
  chain prerouting {
      type filter hook prerouting priority 0;
      tcp dport 21 ct helper set "ftp-standard"
  }
}
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN751"></a>

### Counter

**counter** [packets bytes]

<div class="table"><a name="AEN757"></a>

**Table 7\. Counter specifications**

| Keyword | Description | Type |
| --- | --- | --- |
| packets | initial count of packets | unsigned integer (64 bit) |
| bytes | initial count of bytes | unsigned integer (64 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN777"></a>

### Quota

**quota** [over | until] [used]

<div class="table"><a name="AEN786"></a>

**Table 8\. Quota specifications**

| Keyword | Description | Type |
| --- | --- | --- |
| quota | quota limit, used as the quota name | Two arguments, unsigned interger (64 bit) and string: bytes, kbytes, mbytes. "over" and "until" go before these arguments |
| used | initial value of used quota | Two arguments, unsigned interger (64 bit) and string: bytes, kbytes, mbytes |

</div>

</div>

</div>


## Expressions

Expressions represent values, either constants like network addresses, port numbers etc. or data gathered from the packet during ruleset evaluation. Expressions can be combined using binary, logical, relational and other types of expressions to form complex or relational (match) expressions. They are also used as arguments to certain types of operations, like NAT, packet marking etc.

Each expression has a data type, which determines the size, parsing and representation of symbolic values and type compatibility with other expressions.

<div class="refsect2"><a name="AEN810"></a>

### describe command

**describe** {<tt class="replaceable">_expression_</tt>}

The **describe** command shows information about the type of an expression and its data type.

<div class="example"><a name="AEN819"></a>

**Example 3\. The **describe** command**

| 

<pre class="programlisting">$ nft describe tcp flags
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
                </pre>

 |

</div>

</div>

</div>


## Data types

Data types determine the size, parsing and representation of symbolic values and type compatibility of expressions. A number of global data types exist, in addition some expression types define further data types specific to the expression type. Most data types have a fixed size, some however may have a dynamic size, f.i. the string type.

Types may be derived from lower order types, f.i. the IPv4 address type is derived from the integer type, meaning an IPv4 address can also be specified as an integer value.

In certain contexts (set and map definitions) it is necessary to explicitly specify a data type. Each type has a name which is used for this.

<div class="refsect2"><a name="AEN828"></a>

### Integer type

<div class="table"><a name="AEN831"></a>

**Table 9\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Integer | integer | variable | - |

</div>

The integer type is used for numeric values. It may be specified as decimal, hexadecimal or octal number. The integer type doesn't have a fixed size, its size is determined by the expression for which it is used.

</div>

<div class="refsect2"><a name="AEN850"></a>

### Bitmask type

<div class="table"><a name="AEN853"></a>

**Table 10\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Bitmask | bitmask | variable | integer |

</div>

The bitmask type (**bitmask**) is used for bitmasks.

</div>

<div class="refsect2"><a name="AEN873"></a>

### String type

<div class="table"><a name="AEN876"></a>

**Table 11\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| String | string | variable | - |

</div>

The string type is used to for character strings. A string begins with an alphabetic character (a-zA-Z) followed by zero or more alphanumeric characters or the characters <tt class="literal">/</tt>, <tt class="literal">-</tt>, <tt class="literal">_</tt> and <tt class="literal">.</tt>. In addition anything enclosed in double quotes (<tt class="literal">"</tt>) is recognized as a string.

<div class="example"><a name="AEN900"></a>

**Example 4\. String specification**

| 

<pre class="programlisting"># Interface name
filter input iifname eth0

# Weird interface name
filter input iifname "(eth0)"
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN903"></a>

### Link layer address type

<div class="table"><a name="AEN906"></a>

**Table 12\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Link layer address | lladdr | variable | integer |

</div>

The link layer address type is used for link layer addresses. Link layer addresses are specified as a variable amount of groups of two hexadecimal digits separated using colons (<tt class="literal">:</tt>).

<div class="example"><a name="AEN926"></a>

**Example 5\. Link layer address specification**

| 

<pre class="programlisting"># Ethernet destination MAC address
filter input ether daddr 20:c9:d0:43:12:d9
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN929"></a>

### IPv4 address type

<div class="table"><a name="AEN932"></a>

**Table 13\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| IPv4 address | ipv4_addr | 32 bit | integer |

</div>

The IPv4 address type is used for IPv4 addresses. Addresses are specified in either dotted decimal, dotted hexadecimal, dotted octal, decimal, hexadecimal, octal notation or as a host name. A host name will be resolved using the standard system resolver.

<div class="example"><a name="AEN951"></a>

**Example 6\. IPv4 address specification**

| 

<pre class="programlisting"># dotted decimal notation
filter output ip daddr 127.0.0.1

# host name
filter output ip daddr localhost
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN954"></a>

### IPv6 address type

<div class="table"><a name="AEN957"></a>

**Table 14\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| IPv6 address | ipv6_addr | 128 bit | integer |

</div>

The IPv6 address type is used for IPv6 addresses. FIXME

<div class="example"><a name="AEN976"></a>

**Example 7\. IPv6 address specification**

| 

<pre class="programlisting"># abbreviated loopback address
filter output ip6 daddr ::1
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN979"></a>

### Boolean type

<div class="table"><a name="AEN982"></a>

**Table 15\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| Boolean | boolean | 1 bit | integer |

</div>

The boolean type is a syntactical helper type in user space. It's use is in the right-hand side of a (typically implicit) relational expression to change the expression on the left-hand side into a boolean check (usually for existence).

The following keywords will automatically resolve into a boolean type with given value:

<div class="table"><a name="AEN1002"></a>

**Table 16\.**

| Keyword | Value |
| --- | --- |
| exists | 1 |
| missing | 0 |

</div>

<div class="example"><a name="AEN1017"></a>

**Example 8\. Boolean specification**

The following expressions support a boolean comparison:

<div class="table"><a name="AEN1020"></a>

**Table 17\.**

| Expression | Behaviour |
| --- | --- |
| fib | Check route existence. |
| exthdr | Check IPv6 extension header existence. |
| tcp option | Check TCP option header existence. |

</div>

| 

<pre class="programlisting"># match if route exists
filter input fib daddr . iif oif exists

# match only non-fragmented packets in IPv6 traffic
filter input exthdr frag missing

# match if TCP timestamp option is present
filter input tcp option timestamp exists
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN1039"></a>

### ICMP Type type

<div class="table"><a name="AEN1042"></a>

**Table 18\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| ICMP Type | icmp_type | 8 bit | integer |

</div>

The ICMP Type type is used to conveniently specify the ICMP header's type field.

The following keywords may be used when specifying the ICMP type:

<div class="table"><a name="AEN1062"></a>

**Table 19\.**

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

</div>

<div class="example"><a name="AEN1116"></a>

**Example 9\. ICMP Type specification**

| 

<pre class="programlisting"># match ping packets
filter output icmp type { echo-request, echo-reply }
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN1119"></a>

### ICMPv6 Type type

<div class="table"><a name="AEN1122"></a>

**Table 20\.**

| Name | Keyword | Size | Base type |
| --- | --- | --- | --- |
| ICMPv6 Type | icmpv6_type | 8 bit | integer |

</div>

The ICMPv6 Type type is used to conveniently specify the ICMPv6 header's type field.

The following keywords may be used when specifying the ICMPv6 type:

<div class="table"><a name="AEN1142"></a>

**Table 21\.**

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

</div>

<div class="example"><a name="AEN1208"></a>

**Example 10\. ICMPv6 Type specification**

| 

<pre class="programlisting"># match ICMPv6 ping packets
filter output icmpv6 type { echo-request, echo-reply }
                </pre>

 |

</div>

</div>

</div>


## Primary expressions

The lowest order expression is a primary expression, representing either a constant or a single datum from a packet's payload, meta data or a stateful module.

<div class="refsect2"><a name="AEN1214"></a>

### Meta expressions

**meta** {length | nfproto | l4proto | protocol | priority}

[meta] {mark | iif | iifname | iiftype | oif | oifname | oiftype | skuid | skgid | nftrace | rtclassid | ibriport | obriport | pkttype | cpu | iifgroup | oifgroup | cgroup | random}

A meta expression refers to meta data associated with a packet.

There are two types of meta expressions: unqualified and qualified meta expressions. Qualified meta expressions require the **meta** keyword before the meta key, unqualified meta expressions can be specified by using the meta key directly or as qualified meta expressions.

<div class="table"><a name="AEN1251"></a>

**Table 22\. Meta expression types**

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

</div>

<div class="table"><a name="AEN1348"></a>

**Table 23\. Meta expression specific types**

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

</div>

<div class="example"><a name="AEN1383"></a>

**Example 11\. Using meta expressions**

| 

<pre class="programlisting"># qualified meta expression
filter output meta oif eth0

# unqualified meta expression
filter output oif eth0
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN1386"></a>

### fib expressions

**fib** {saddr | daddr [mark | iif | oif]} {oif | oifname | type}

A fib expression queries the fib (forwarding information base) to obtain information such as the output interface index a particular address would use. The input is a tuple of elements that is used as input to the fib lookup functions.

<div class="table"><a name="AEN1404"></a>

**Table 24\. fib expression specific types**

| Keyword | Description | Type |
| --- | --- | --- |
| oif | Output interface index | integer (32 bit) |
| oifname | Output interface name | string |
| type | Address type | fib_addrtype |

</div>

<div class="example"><a name="AEN1429"></a>

**Example 12\. Using fib expressions**

| 

<pre class="programlisting"># drop packets without a reverse path
filter prerouting fib saddr . iif oif missing drop

# drop packets to address not configured on ininterface
filter prerouting fib daddr . iif type != { local, broadcast, multicast } drop

# perform lookup in a specific 'blackhole' table (0xdead, needs ip appropriate ip rule)
filter prerouting meta mark set 0xdead fib daddr . mark type vmap { blackhole : drop, prohibit : jump prohibited, unreachable : drop }
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN1432"></a>

### Routing expressions

**rt** {classid | nexthop}

A routing expression refers to routing data associated with a packet.

<div class="table"><a name="AEN1442"></a>

**Table 25\. Routing expression types**

| Keyword | Description | Type |
| --- | --- | --- |
| classid | Routing realm | realm |
| nexthop | Routing nexthop | ipv4_addr/ipv6_addr |

</div>

<div class="table"><a name="AEN1463"></a>

**Table 26\. Routing expression specific types**

| Type | Description |
| --- | --- |
| realm | Routing Realm (32 bit number). Can be specified numerically or as symbolic name defined in /etc/iproute2/rt_realms. |

</div>

<div class="example"><a name="AEN1477"></a>

**Example 13\. Using routing expressions**

| 

<pre class="programlisting"># IP family independent rt expression
filter output rt classid 10

# IP family dependent rt expressions
ip filter output rt nexthop 192.168.0.1
ip6 filter output rt nexthop fd00::1
inet filter meta nfproto ipv4 output rt nexthop 192.168.0.1
inet filter meta nfproto ipv6 output rt nexthop fd00::1
                    </pre>

 |

</div>

</div>

</div>


## Payload expressions

Payload expressions refer to data from the packet's payload.

<div class="refsect2"><a name="AEN1483"></a>

### Ethernet header expression

**ether** [<tt class="replaceable">_ethernet header field_</tt>]

<div class="table"><a name="AEN1491"></a>

**Table 27\. Ethernet header expression types**

| Keyword | Description | Type |
| --- | --- | --- |
| daddr | Destination MAC address | ether_addr |
| saddr | Source MAC address | ether_addr |
| type | EtherType | ether_type |

</div>

</div>

<div class="refsect2"><a name="AEN1515"></a>

### VLAN header expression

**vlan** [<tt class="replaceable">_VLAN header field_</tt>]

<div class="table"><a name="AEN1523"></a>

**Table 28\. VLAN header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| id | VLAN ID (VID) | integer (12 bit) |
| cfi | Canonical Format Indicator | integer (1 bit) |
| pcp | Priority code point | integer (3 bit) |
| type | EtherType | ether_type |

</div>

</div>

<div class="refsect2"><a name="AEN1551"></a>

### ARP header expression

**arp** [<tt class="replaceable">_ARP header field_</tt>]

<div class="table"><a name="AEN1559"></a>

**Table 29\. ARP header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| htype | ARP hardware type | integer (16 bit) |
| ptype | EtherType | ether_type |
| hlen | Hardware address len | integer (8 bit) |
| plen | Protocol address len | integer (8 bit) |
| operation | Operation | arp_op |

</div>

</div>

<div class="refsect2"><a name="AEN1591"></a>

### IPv4 header expression

**ip** [<tt class="replaceable">_IPv4 header field_</tt>]

<div class="table"><a name="AEN1599"></a>

**Table 30\. IPv4 header expression**

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

</div>

</div>

<div class="refsect2"><a name="AEN1659"></a>

### ICMP header expression

**icmp** [<tt class="replaceable">_ICMP header field_</tt>]

<div class="table"><a name="AEN1667"></a>

**Table 31\. ICMP header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| type | ICMP type field | icmp_type |
| code | ICMP code field | integer (8 bit) |
| checksum | ICMP checksum field | integer (16 bit) |
| id | ID of echo request/response | integer (16 bit) |
| sequence | sequence number of echo request/response | integer (16 bit) |
| gateway | gateway of redirects | integer (32 bit) |
| mtu | MTU of path MTU discovery | integer (16 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN1707"></a>

### IPv6 header expression

**ip6** [<tt class="replaceable">_IPv6 header field_</tt>]

<div class="table"><a name="AEN1715"></a>

**Table 32\. IPv6 header expression**

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

</div>

</div>

<div class="refsect2"><a name="AEN1763"></a>

### ICMPv6 header expression

**icmpv6** [<tt class="replaceable">_ICMPv6 header field_</tt>]

<div class="table"><a name="AEN1771"></a>

**Table 33\. ICMPv6 header expression**

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

</div>

</div>

<div class="refsect2"><a name="AEN1815"></a>

### TCP header expression

**tcp** [<tt class="replaceable">_TCP header field_</tt>]

<div class="table"><a name="AEN1823"></a>

**Table 34\. TCP header expression**

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

</div>

</div>

<div class="refsect2"><a name="AEN1875"></a>

### UDP header expression

**udp** [<tt class="replaceable">_UDP header field_</tt>]

<div class="table"><a name="AEN1883"></a>

**Table 35\. UDP header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| length | Total packet length | integer (16 bit) |
| checksum | Checksum | integer (16 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN1911"></a>

### UDP-Lite header expression

**udplite** [<tt class="replaceable">_UDP-Lite header field_</tt>]

<div class="table"><a name="AEN1919"></a>

**Table 36\. UDP-Lite header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| checksum | Checksum | integer (16 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN1943"></a>

### SCTP header expression

**sctp** [<tt class="replaceable">_SCTP header field_</tt>]

<div class="table"><a name="AEN1951"></a>

**Table 37\. SCTP header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |
| vtag | Verfication Tag | integer (32 bit) |
| checksum | Checksum | integer (32 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN1979"></a>

### DCCP header expression

**dccp** [<tt class="replaceable">_DCCP header field_</tt>]

<div class="table"><a name="AEN1987"></a>

**Table 38\. DCCP header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| sport | Source port | inet_service |
| dport | Destination port | inet_service |

</div>

</div>

<div class="refsect2"><a name="AEN2007"></a>

### Authentication header expression

**ah** [<tt class="replaceable">_AH header field_</tt>]

<div class="table"><a name="AEN2015"></a>

**Table 39\. AH header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| nexthdr | Next header protocol | inet_proto |
| hdrlength | AH Header length | integer (8 bit) |
| reserved | Reserved area | integer (16 bit) |
| spi | Security Parameter Index | integer (32 bit) |
| sequence | Sequence number | integer (32 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN2047"></a>

### Encrypted security payload header expression

**esp** [<tt class="replaceable">_ESP header field_</tt>]

<div class="table"><a name="AEN2055"></a>

**Table 40\. ESP header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| spi | Security Parameter Index | integer (32 bit) |
| sequence | Sequence number | integer (32 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN2075"></a>

### IPcomp header expression

**comp** [<tt class="replaceable">_IPComp header field_</tt>]

<div class="table"><a name="AEN2083"></a>

**Table 41\. IPComp header expression**

| Keyword | Description | Type |
| --- | --- | --- |
| nexthdr | Next header protocol | inet_proto |
| flags | Flags | bitmask |
| cpi | Compression Parameter Index | integer (16 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN2107"></a>

### Extension header expressions

Extension header expressions refer to data from variable-sized protocol headers, such as IPv6 extension headers and TCPs options.

nftables currently supports matching (finding) a given ipv6 extension header or TCP option.

**hbh** {nexthdr | hdrlength}

**frag** {nexthdr | frag-off | more-fragments | id}

**rt** {nexthdr | hdrlength | type | seg-left}

**dst** {nexthdr | hdrlength}

**mh** {nexthdr | hdrlength | checksum | type}

**tcp option** {eol | noop | maxseg | window | sack-permitted | sack | sack0 | sack1 | sack2 | sack3 | timestamp} [<tt class="replaceable">_tcp_option_field_</tt>]

The following syntaxes are valid only in a relational expression with boolean type on right-hand side for checking header existence only:

**exthdr** {hbh | frag | rt | dst | mh}

**tcp option** {eol | noop | maxseg | window | sack-permitted | sack | sack0 | sack1 | sack2 | sack3 | timestamp}

<div class="table"><a name="AEN2182"></a>

**Table 42\. IPv6 extension headers**

| Keyword | Description |
| --- | --- |
| hbh | Hop by Hop |
| rt | Routing Header |
| frag | Fragmentation header |
| dst | dst options |
| mh | Mobility Header |

</div>

<div class="table"><a name="AEN2207"></a>

**Table 43\. TCP Options**

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

</div>

<div class="example"><a name="AEN2264"></a>

**Example 14\. finding TCP options**

| 

<pre class="programlisting">filter input tcp option sack-permitted kind 1 counter
                    </pre>

 |

</div>

<div class="example"><a name="AEN2267"></a>

**Example 15\. matching IPv6 exthdr**

| 

<pre class="programlisting">ip6 filter input frag more-fragments 1 counter
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN2270"></a>

### Conntrack expressions

Conntrack expressions refer to meta data of the connection tracking entry associated with a packet.

There are three types of conntrack expressions. Some conntrack expressions require the flow direction before the conntrack key, others must be used directly because they are direction agnostic. The **packets**, **bytes** and **avgpkt** keywords can be used with or without a direction. If the direction is omitted, the sum of the original and the reply direction is returned. The same is true for the **zone**, if a direction is given, the zone is only matched if the zone id is tied to the given direction.

**ct** {state | direction | status | mark | expiration | helper | label | l3proto | protocol | bytes | packets | avgpkt | zone}

**ct** {original | reply} {l3proto | protocol | saddr | daddr | proto-src | proto-dst | bytes | packets | avgpkt | zone}

<div class="table"><a name="AEN2312"></a>

**Table 44\. Conntrack expressions**

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
| bytes | bytecount seen, see description for **packets** keyword | integer (64 bit) |
| avgpkt | average bytes per packet, see description for **packets** keyword | integer (64 bit) |
| zone | conntrack zone | integer (16 bit) |

</div>

</div>

</div>


## Statements

Statements represent actions to be performed. They can alter control flow (return, jump to a different chain, accept or drop the packet) or can perform actions, such as logging, rejecting a packet, etc.

Statements exist in two kinds. Terminal statements unconditionally terminate evaluation of the current rule, non-terminal statements either only conditionally or never terminate evaluation of the current rule, in other words, they are passive from the ruleset evaluation perspective. There can be an arbitrary amount of non-terminal statements in a rule, but only a single terminal statement as the final statement.

<div class="refsect2"><a name="AEN2398"></a>

### Verdict statement

The verdict statement alters control flow in the ruleset and issues policy decisions for packets.

{accept | drop | queue | continue | return}

{jump | goto} {<tt class="replaceable">_chain_</tt>}

<div class="variablelist">

<dl>

<dt>`accept`</dt>

<dd>

Terminate ruleset evaluation and accept the packet.

</dd>

<dt>`drop`</dt>

<dd>

Terminate ruleset evaluation and drop the packet.

</dd>

<dt>`queue`</dt>

<dd>

Terminate ruleset evaluation and queue the packet to userspace.

</dd>

<dt>`continue`</dt>

<dd>

Continue ruleset evaluation with the next rule. FIXME

</dd>

<dt>`return`</dt>

<dd>

Return from the current chain and continue evaluation at the next rule in the last chain. If issued in a base chain, it is equivalent to **accept**.

</dd>

<dt>`jump <tt class="replaceable">_chain_</tt>`</dt>

<dd>

Continue evaluation at the first rule in <tt class="replaceable">_chain_</tt>. The current position in the ruleset is pushed to a call stack and evaluation will continue there when the new chain is entirely evaluated of a **return** verdict is issued.

</dd>

<dt>`goto <tt class="replaceable">_chain_</tt>`</dt>

<dd>

Similar to **jump**, but the current position is not pushed to the call stack, meaning that after the new chain evaluation will continue at the last chain instead of the one containing the goto statement.

</dd>

</dl>

</div>

<div class="example"><a name="AEN2459"></a>

**Example 16\. Verdict statements**

| 

<pre class="programlisting"># process packets from eth0 and the internal network in from_lan
# chain, drop all packets from eth0 with different source addresses.

filter input iif eth0 ip saddr 192.168.0.0/24 jump from_lan
filter input iif eth0 drop
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN2462"></a>

### Payload statement

The payload statement alters packet content. It can be used for example to set ip DSCP (differv) header field or ipv6 flow labels.

<div class="example"><a name="AEN2466"></a>

**Example 17\. route some packets instead of bridging**

| 

<pre class="programlisting"># redirect tcp:http from 192.160.0.0/16 to local machine for routing instead of bridging
# assumes 00:11:22:33:44:55 is local MAC address.
bridge input meta iif eth0 ip saddr 192.168.0.0/16 tcp dport 80 meta pkttype set unicast ether daddr set 00:11:22:33:44:55
                    </pre>

 |

</div>

<div class="example"><a name="AEN2469"></a>

**Example 18\. Set IPv4 DSCP header field**

| 

<pre class="programlisting">ip forward ip dscp set 42
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN2472"></a>

### Log statement

**log** [prefix <tt class="replaceable">_quoted_string_</tt>] [level <tt class="replaceable">_syslog-level_</tt>] [flags <tt class="replaceable">_log-flags_</tt>]

**log** [group <tt class="replaceable">_nflog_group_</tt>] [prefix <tt class="replaceable">_quoted_string_</tt>] [queue-threshold <tt class="replaceable">_value_</tt>] [snaplen <tt class="replaceable">_size_</tt>]

The log statement enables logging of matching packets. When this statement is used from a rule, the Linux kernel will print some information on all matching packets, such as header fields, via the kernel log (where it can be read with dmesg(1) or read in the syslog). If the group number is specified, the Linux kernel will pass the packet to nfnetlink_log which will multicast the packet through a netlink socket to the specified multicast group. One or more userspace processes may subscribe to the group to receive the packets, see libnetfilter_queue documentation for details. This is a non-terminating statement, so the rule evaluation continues after the packet is logged.

<div class="table"><a name="AEN2495"></a>

**Table 45\. log statement options**

| Keyword | Description | Type |
| --- | --- | --- |
| prefix | Log message prefix | quoted string |
| syslog-level | Syslog level of logging | string: emerg, alert, crit, err, warn [default], notice, info, debug |
| group | NFLOG group to send messages to | unsigned integer (16 bit) |
| snaplen | Length of packet payload to include in netlink message | unsigned integer (32 bit) |
| queue-threshold | Number of packets to queue inside the kernel before sending them to userspace | unsigned integer (32 bit) |

</div>

<div class="table"><a name="AEN2527"></a>

**Table 46\. log-flags**

| Flag | Description |
| --- | --- |
| tcp sequence | Log TCP sequence numbers. |
| tcp options | Log options from the TCP packet header. |
| ip options | Log options from the IP/IPv6 packet header. |
| skuid | Log the userid of the process which generated the packet. |
| ether | Decode MAC addresses and protocol. |
| all | Enable all log flags listed above. |

</div>

<div class="example"><a name="AEN2556"></a>

**Example 19\. Using log statement**

| 

<pre class="programlisting"># log the UID which generated the packet and ip options
ip filter output log flags skuid flags ip options

# log the tcp sequence numbers and tcp options from the TCP packet
ip filter output log flags tcp sequence,options

# enable all supported log flags
ip6 filter output log flags all
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN2559"></a>

### Reject statement

**reject** [with] {icmp | icmp6 | icmpx} [type] {icmp_type | icmp6_type | icmpx_type}

**reject** [with] {tcp} {reset}

A reject statement is used to send back an error packet in response to the matched packet otherwise it is equivalent to drop so it is a terminating statement, ending rule traversal. This statement is only valid in the input, forward and output chains, and user-defined chains which are only called from those chains.

<div class="table"><a name="AEN2580"></a>

**Table 47\. reject statement type (ip)**

| Value | Description | Type |
| --- | --- | --- |
| icmp_type | ICMP type response to be sent to the host | net-unreachable, host-unreachable, prot-unreachable, port-unreachable [default], net-prohibited, host-prohibited, admin-prohibited |

</div>

<div class="table"><a name="AEN2596"></a>

**Table 48\. reject statement type (ip6)**

| Value | Description | Type |
| --- | --- | --- |
| icmp6_type | ICMPv6 type response to be sent to the host | no-route, admin-prohibited, addr-unreachable, port-unreachable [default], policy-fail, reject-route |

</div>

<div class="table"><a name="AEN2612"></a>

**Table 49\. reject statement type (inet)**

| Value | Description | Type |
| --- | --- | --- |
| icmpx_type | ICMPvXtype abstraction response to be sent to the host, this is a set of types that overlap in IPv4 and IPv6 to be used from the inet family. | port-unreachable [default], admin-prohibited, no-route, host-unreachable |

</div>

</div>

<div class="refsect2"><a name="AEN2628"></a>

### Counter statement

A counter statement sets the hit count of packets along with the number of bytes.

**counter** {packets <tt class="replaceable">_number_</tt> } {bytes <tt class="replaceable">_number_</tt> }

</div>

<div class="refsect2"><a name="AEN2638"></a>

### Conntrack statement

The conntrack statement can be used to set the conntrack mark and conntrack labels.

**ct** {mark | eventmask | label | zone} [set]<tt class="replaceable">_value_</tt>

The ct statement sets meta data associated with a connection. The zone id has to be assigned before a conntrack lookup takes place, i.e. this has to be done in prerouting and possibly output (if locally generated packets need to be placed in a distinct zone), with a hook priority of -300.

<div class="table"><a name="AEN2653"></a>

**Table 50\. Conntrack statement types**

| Keyword | Description | Value |
| --- | --- | --- |
| eventmask | conntrack event bits | bitmask, integer (32 bit) |
| helper | name of ct helper object to assign to the connection | quoted string |
| mark | Connection tracking mark | mark |
| label | Connection tracking label | label |
| zone | conntrack zone | integer (16 bit) |

</div>

<div class="example"><a name="AEN2686"></a>

**Example 20\. save packet nfmark in conntrack**

| 

<pre class="programlisting">ct mark set meta mark
                    </pre>

 |

</div>

<div class="example"><a name="AEN2689"></a>

**Example 21\. set zone mapped via interface**

| 

<pre class="programlisting">table inet raw {
  chain prerouting {
      type filter hook prerouting priority -300;
      ct zone set iif map { "eth1" : 1, "veth1" : 2 }
  }
  chain output {
      type filter hook output priority -300;
      ct zone set oif map { "eth1" : 1, "veth1" : 2 }
  }
}
                </pre>

 |

</div>

<div class="example"><a name="AEN2692"></a>

**Example 22\. restrict events reported by ctnetlink**

| 

<pre class="programlisting">ct eventmask set new or related or destroy
                </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN2695"></a>

### Meta statement

A meta statement sets the value of a meta expression. The existing meta fields are: priority, mark, pkttype, nftrace.

**meta** {mark | priority | pkttype | nftrace} [set]<tt class="replaceable">_value_</tt>

A meta statement sets meta data associated with a packet.

<div class="table"><a name="AEN2710"></a>

**Table 51\. Meta statement types**

| Keyword | Description | Value |
| --- | --- | --- |
| priority | TC packet priority | tc_handle |
| mark | Packet mark | mark |
| pkttype | packet type | pkt_type |
| nftrace | ruleset packet tracing on/off. Use **monitor trace** command to watch traces | 0, 1 |

</div>

</div>

<div class="refsect2"><a name="AEN2739"></a>

### Limit statement

**limit** [rate] [over]<tt class="replaceable">_packet_number_</tt> [/] {second | minute | hour | day} [burst <tt class="replaceable">_packet_number_</tt> packets]

**limit** [rate] [over]<tt class="replaceable">_byte_number_</tt> {bytes | kbytes | mbytes} [/] {second | minute | hour | day | week} [burst <tt class="replaceable">_byte_number_</tt> bytes]

A limit statement matches at a limited rate using a token bucket filter. A rule using this statement will match until this limit is reached. It can be used in combination with the log statement to give limited logging. The **over** keyword, that is optional, makes it match over the specified rate.

<div class="table"><a name="AEN2775"></a>

**Table 52\. limit statement values**

| Value | Description | Type |
| --- | --- | --- |
| packet_number | Number of packets | unsigned integer (32 bit) |
| byte_number | Number of bytes | unsigned integer (32 bit) |

</div>

</div>

<div class="refsect2"><a name="AEN2795"></a>

### NAT statements

**snat** [to <tt class="replaceable">_address_</tt> [:port]] [persistent, random, fully-random]

**snat** [to <tt class="replaceable">_address_</tt> - <tt class="replaceable">_address_</tt> [:<tt class="replaceable">_port_</tt> - <tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

**dnat** [to <tt class="replaceable">_address_</tt> [:<tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

**dnat** [to <tt class="replaceable">_address_</tt> [:<tt class="replaceable">_port_</tt> - <tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

**masquerade** [to [:<tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

**masquerade** [to [:<tt class="replaceable">_port_</tt> - <tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

**redirect** [to [:<tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

**redirect** [to [:<tt class="replaceable">_port_</tt> - <tt class="replaceable">_port_</tt>]] [persistent, random, fully-random]

The nat statements are only valid from nat chain types.

The **snat** and **masquerade** statements specify that the source address of the packet should be modified. While **snat** is only valid in the postrouting and input chains, **masquerade** makes sense only in postrouting. The **dnat** and **redirect** statements are only valid in the prerouting and output chains, they specify that the destination address of the packet should be modified. You can use non-base chains which are called from base chains of nat chain type too. All future packets in this connection will also be mangled, and rules should cease being examined.

The **masquerade** statement is a special form of **snat** which always uses the outgoing interface's IP address to translate to. It is particularly useful on gateways with dynamic (public) IP addresses.

The **redirect** statement is a special form of **dnat** which always translates the destination address to the local host's one. It comes in handy if one only wants to alter the destination port of incoming traffic on different interfaces.

Note that all nat statements require both prerouting and postrouting base chains to be present since otherwise packets on the return path won't be seen by netfilter and therefore no reverse translation will take place.

<div class="table"><a name="AEN2870"></a>

**Table 53\. NAT statement values**

| Expression | Description | Type |
| --- | --- | --- |
| address | Specifies that the source/destination address of the packet should be modified. You may specify a mapping to relate a list of tuples composed of arbitrary expression key with address value. | ipv4_addr, ipv6_addr, eg. abcd::1234, or you can use a mapping, eg. meta mark map { 10 : 192.168.1.2, 20 : 192.168.1.3 } |
| port | Specifies that the source/destination address of the packet should be modified. | port number (16 bits) |

</div>

<div class="table"><a name="AEN2890"></a>

**Table 54\. NAT statement flags**

| Flag | Description |
| --- | --- |
| persistent | Gives a client the same source-/destination-address for each connection. |
| random | If used then port mapping will be randomized using a random seeded MD5 hash mix using source and destination address and destination port. |
| fully-random | If used then port mapping is generated based on a 32-bit pseudo-random algorithm. |

</div>

<div class="example"><a name="AEN2910"></a>

**Example 23\. Using NAT statements**

| 

<pre class="programlisting"># create a suitable table/chain setup for all further examples
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
                    </pre>

 |

</div>

</div>

<div class="refsect2"><a name="AEN2913"></a>

### Queue statement

This statement passes the packet to userspace using the nfnetlink_queue handler. The packet is put into the queue identified by its 16-bit queue number. Userspace can inspect and modify the packet if desired. Userspace must then drop or reinject the packet into the kernel. See libnetfilter_queue documentation for details.

**queue** [num <tt class="replaceable">_queue_number_</tt>] [bypass]

**queue** [num <tt class="replaceable">_queue_number_from_</tt> - <tt class="replaceable">_queue_number_to_</tt>] [bypass,fanout]

<div class="table"><a name="AEN2929"></a>

**Table 55\. queue statement values**

| Value | Description | Type |
| --- | --- | --- |
| queue_number | Sets queue number, default is 0. | unsigned integer (16 bit) |
| queue_number_from | Sets initial queue in the range, if fanout is used. | unsigned integer (16 bit) |
| queue_number_to | Sets closing queue in the range, if fanout is used. | unsigned integer (16 bit) |

</div>

<div class="table"><a name="AEN2953"></a>

**Table 56\. queue statement flags**

| Flag | Description |
| --- | --- |
| bypass | Let packets go through if userspace application cannot back off. Before using this flag, read libnetfilter_queue documentation for performance tuning recomendations. |
| fanout | Distribute packets between several queues. |

</div>

</div>

</div>


## Additional commands

These are some additional commands included in nft.

<div class="refsect2"><a name="AEN2972"></a>

### export

Export your current ruleset in XML or JSON format to stdout.

Examples:

| 

<pre class="programlisting">% nft export xml
[...]
% nft export json
[...]
                </pre>

 |

</div>

<div class="refsect2"><a name="AEN2977"></a>

### monitor

The monitor command allows you to listen to Netlink events produced by the nf_tables subsystem, related to creation and deletion of objects. When they ocurr, nft will print to stdout the monitored events in either XML, JSON or native nft format.

To filter events related to a concrete object, use one of the keywords 'tables', 'chains', 'sets', 'rules', 'elements'.

To filter events related to a concrete action, use keyword 'new' or 'destroy'.

Hit ^C to finish the monitor operation.

<div class="example"><a name="AEN2983"></a>

**Example 24\. Listen to all events, report in native nft format**

| 

<pre class="programlisting">% nft monitor
                </pre>

 |

</div>

<div class="example"><a name="AEN2986"></a>

**Example 25\. Listen to added tables, report in XML format**

| 

<pre class="programlisting">% nft monitor new tables xml
                </pre>

 |

</div>

<div class="example"><a name="AEN2989"></a>

**Example 26\. Listen to deleted rules, report in JSON format**

| 

<pre class="programlisting">% nft monitor destroy rules json
                </pre>

 |

</div>

<div class="example"><a name="AEN2992"></a>

**Example 27\. Listen to both new and destroyed chains, in native nft format**

| 

<pre class="programlisting">% nft monitor chains
                </pre>

 |

</div>

</div>

</div>


## Error reporting

When an error is detected, nft shows the line(s) containing the error, the position of the erroneous parts in the input stream and marks up the erroneous parts using carrets (<tt class="literal">^</tt>). If the error results from the combination of two expressions or statements, the part imposing the constraints which are violated is marked using tildes (<tt class="literal">~</tt>).

For errors returned by the kernel, nft can't detect which parts of the input caused the error and the entire command is marked.

<div class="example"><a name="AEN3001"></a>

**Example 28\. Error caused by single incorrect expression**

| 

<pre class="programlisting"><cmdline>:1:19-22: Error: Interface does not exist
filter output oif eth0
                  ^^^^
            </pre>

 |

</div>

<div class="example"><a name="AEN3004"></a>

**Example 29\. Error caused by invalid combination of two expressions**

| 

<pre class="programlisting"><cmdline>:1:28-36: Error: Right hand side of relational expression (==) must be constant
filter output tcp dport == tcp dport
                        ~~ ^^^^^^^^^
            </pre>

 |

</div>

<div class="example"><a name="AEN3007"></a>

**Example 30\. Error returned by the kernel**

| 

<pre class="programlisting"><cmdline>:0:0-23: Error: Could not process rule: Operation not permitted
filter output oif wlan0
^^^^^^^^^^^^^^^^^^^^^^^
            </pre>

 |

</div>

</div>


## Exit status

On success, nft exits with a status of 0\. Unspecified errors cause it to exit with a status of 1, memory allocation errors with a status of 2, unable to open Netlink socket with 3.

</div>


## See Also

iptables(8), ip6tables(8), arptables(8), ebtables(8), ip(8), tc(8)

There is an official wiki at: http://wiki.nftables.org

</div>


## Authors

nftables was written by Patrick McHardy and Pablo Neira Ayuso, among many other contributors from the Netfilter community.

</div>


## Copyright

Copyright © 2008-2014 Patrick McHardy `<[kaber@trash.net](mailto:kaber@trash.net)>`
Copyright © 2013-2016 Pablo Neira Ayuso `<[pablo@netfilter.org](mailto:pablo@netfilter.org)>`

nftables is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License version 2 as published by the Free Software Foundation.

This documentation is licenced under the terms of the Creative Commons Attribution-ShareAlike 4.0 license, [CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/).

</div>
