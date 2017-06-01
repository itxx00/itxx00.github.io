---
layout: post
title: "OpenVPN+MySQL环境搭建记录"
description: ""
categories: [system]
tags: [openvpn, freeradius]
---

> 本文记录了在centos5.5系统中搭建openvpn+mysql+freeradius的过程

* Kramdown table of contents
{:toc .toc}

系统环境：centos5.5

Eth0：192.168.0.2

校准系统时间：

```
ntpdate ntp.api.bz
hwclock -w
```

所需软件包：

```
yum install openssl openssl-devel gcc gcc-c++ mysql mysql-devel mysql-server php php-gd php-devel php-pear php-pear-DB php-mysql php-pdo php-cli php-mbstring php-mcrypt httpd
```

下载源码包：

```
wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.05.tar.gz
wget http://swupdate.openvpn.net/community/releases/openvpn-2.2.0.tar.gz
wget ftp://ftp.freeradius.org/pub/freeradius/freeradius-server-2.1.10.tar.gz
wget http://sourceforge.net/projects/daloradius/files/daloradius/daloradius0.9-9/daloradius-0.9-9-rc1.tar.gz/download
wget http://www.nongnu.org/radiusplugin/radiusplugin_v2.1a_beta1.tar.gz
wget http://sourceforge.net/projects/phpmyadmin/files%2FphpMyAdmin%2F2.11.11.3%2FphpMyAdmin-2.11.11.3-all-languages.tar.bz2/download
```

1.安装lzo-2.0.5  使openvpn支持压缩功能

```
tar xf lzo-2.05.tar.gz &&cd lzo-*
./configure
make && make install
cd ..
```

2.安装openvpn服务

```
tar xf openvpn-2.2.0.tar.gz &&cd openvpn-*
./configure
make && make install
```

复制证书生成所需文件

```
mkdir /etc/openvpn && cp easy-rsa /etc/openvpn -r
```

3.安装radiusplugin

```
tar xf radiusplugin_v2.1a_beta1.tar.gz && cd radiusplugin_*
make #将会生成radiusplugin.so 将其cp到/etc/openvpn目录下
cp radiusplugin.cnf /etc/openvpn/conf/
```

4.生成服务端所需证书

```
cd /etc/openvpn/easy-rsa/2.0/
vi vars   #编辑变量
.  vars  #source命令使变量生效
./clean-all
./build-ca  #ca.crt ---Root CA证书，用于签发Server和Client证书
./build-key-server server #server.crt server.key-创建并签发VPN Server使用的CA
./build-key client1 #client1.crt client1.key #如果客户端需要使用证书方式认证则需要这个东东，创建客户端证书，一个客户端一个证书
./build-key client2
./build-dh  #dh1024.pem--生成Diffie-Hellman文件 ,TLS用到
openvpn --genkey --secret keys/ta.key #生成tls auth key

```
复制证书文件到配置文件目录

```
mkdir /etc/openvpn/conf && cd keys
cp ca.crt server.crt server.key dh1024.pem ta.key /etc/openvpn/conf/
tar czf clientkey.tgz ca.crt client1.crt client1.key # 如果客户端使用证书方式认证则这里需要这个东东，本文介绍的radius方式不需要。 
```

5.编辑openvpn服务端配置文件

vi /etc/openvpn/conf/server.conf

```
port 1194
proto udp
dev tap   #TAP设备是一块虚拟的以太网卡，TUN设备是一个虚拟的点到点IP链接。 
ca /etc/openvpn/conf/ca.crt
cert /etc/openvpn/conf/server.crt
key /etc/openvpn/conf/server.key  # This file should be kept secret
dh /etc/openvpn/conf/dh1024.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 10.8.0.1"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"
client-to-client
keepalive 10 120

tls-auth /etc/openvpn/conf/ta.key 0

comp-lzo
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
verb 3

client-to-client #如果让Client之间可以相互看见，去掉本行的注释掉，否则Client之间无法相互访问

;duplicate-cn  #是否允许一个User同时登录多次，去掉本行注释后可以使用同一个用户名登录多次

client-cert-not-required #客户端不使用CA证书验证

username-as-common-name #使用客户提供的UserName作为Common Name

;max-clients 1000  #最大连接数

plugin /etc/openvpn/radiusplugin.so /etc/openvpn/conf/radiusplugin.cnf #radius插件

script-security 3
```

编辑radius插件配置文件

vi /etc/openvpn/conf/radiusplugin.cnf

```
NAS-Identifier=OpenVpn
Service-Type=5
Framed-Protocol=1
NAS-Port-Type=5
NAS-IP-Address=127.0.0.1
OpenVPNConfig=/etc/openvpn/conf/server.conf
subnet=255.255.255.0
overwriteccfiles=true
nonfatalaccounting=false
server
{
        # The UDP port for radius accounting.
        acctport=1813
        # The UDP port for radius authentication.
        authport=1812
        # The name or ip address of the radius server.
        name=127.0.0.1
        # How many times should the plugin send the if there is no response?
        retry=1
        # How long should the plugin wait for a response?
        wait=1
        # The shared secret.
        sharedsecret=testpw
}
```

启动openvpn

```
/usr/local/sbin/openvpn  --cd /etc/openvpn \
 --config /etc/openvpn/conf/server.conf &
```

配置内核路由转发和iptables

```
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -j MASQUERADE
iptables -A INPUT -p udp  --dport 1194 -j ACCEPT
```

6.安装freeradius

```
tar  xf  freeradius-server-2.1.10.tar.bz2
cd  freeradius-*
./configure
make && make install
vi /etc/ld.so.conf
#加入:
/usr/local/lib
#然后
ldconfig
```

启动mysql

```
/etc/init.d/mysqld start
#初始化mysql
mysql_secure_installation
mysql -uroot -p
create database radius;
cd /usr/local/etc/raddb/sql/mysql
mysql -uroot -p radius < schema.sql
mysql -uroot -p < admin.sql
```

配置radiusd
```
cd /usr/local/etc/raddb
vi radiusd.conf
```

```
prefix = /usr/local
exec_prefix = ${prefix}
sysconfdir = ${prefix}/etc
localstatedir = ${prefix}/var
sbindir = ${exec_prefix}/sbin
logdir = ${localstatedir}/log/radius
raddbdir = ${sysconfdir}/raddb
radacctdir = ${logdir}/radacct
name = radiusd
confdir = ${raddbdir}
run_dir = ${localstatedir}/run/${name}
db_dir = ${raddbdir}
libdir = ${exec_prefix}/lib
pidfile = ${run_dir}/${name}.pid
max_request_time = 30
cleanup_delay = 5
max_requests = 1024
listen {
        type = auth
        ipaddr = *
        port = 0
}
listen {
        ipaddr = *
        port = 0
        type = acct
}
hostname_lookups = no
allow_core_dumps = no
regular_expressions     = yes
extended_expressions    = yes
log {
        destination = files
        file = ${logdir}/radius.log
        syslog_facility = daemon
        stripped_names = no
        auth = no
        auth_badpass = no
        auth_goodpass = no
}
checkrad = ${sbindir}/checkrad
security {
        max_attributes = 200
        reject_delay = 1
        status_server = yes
}
proxy_requests  = yes
$INCLUDE proxy.conf
$INCLUDE clients.conf
thread pool {
        start_servers = 5
        max_servers = 32
        min_spare_servers = 3
        max_spare_servers = 10
        max_requests_per_server = 0
}
modules {
        $INCLUDE ${confdir}/modules/
        $INCLUDE eap.conf
        $INCLUDE sql.conf
        $INCLUDE sql/mysql/counter.conf
        $INCLUDE sqlippool.conf
}
instantiate {
        exec
        expr
        expiration
        logintime
}
$INCLUDE policy.conf
$INCLUDE sites-enabled/
```

vi clients.conf

```
client localhost {
        ipaddr = 127.0.0.1
        secret          = testpw
        require_message_authenticator = no
}
client 192.168.0.2 {
        ipaddr = 192.168.0.2
        secret          = testpw
        require_message_authenticator = no
}
```

vi sites-enabled/default

```
authorize {
        preprocess
        chap
        mschap
        digest
        suffix
        eap {
                ok = return
        }
        sql
        expiration
        logintime
        pap
}
authenticate {
        Auth-Type PAP {
                pap
        }
        Auth-Type CHAP {
                chap
        }
        Auth-Type MS-CHAP {
                mschap
        }
        digest
        unix
        eap
}
preacct {
        preprocess
        acct_unique
        suffix
        files
}
accounting {
        detail
        radutmp
        sql
        exec
        attr_filter.accounting_response
}
session {
        radutmp
}
post-auth {
        exec
        Post-Auth-Type REJECT {
                attr_filter.access_reject
        }
}
post-proxy {
        eap
}
```

vi  sql.conf

```
sql {
        database = "mysql"
        driver = "rlm_sql_${database}"
        server = "localhost"
        port = 3306
        login = "radius"
        password = "radpass"
        radius_db = "radius"
        acct_table1 = "radacct"
        acct_table2 = "radacct"
        postauth_table = "radpostauth"
        authcheck_table = "radcheck"
        authreply_table = "radreply"
        groupcheck_table = "radgroupcheck"
        groupreply_table = "radgroupreply"
        usergroup_table = "radusergroup"
        deletestalesessions = yes
        sqltrace = no
        sqltracefile = ${logdir}/sqltrace.sql
        num_sql_socks = 5
        connect_failure_retry_delay = 60
        lifetime = 0
        max_queries = 0
        readclients = yes
        nas_table = "nas"
        $INCLUDE sql/${database}/dialup.conf
}
```

启动radius： `radiusd  &`


安装phpmyadmin管理mysql
```
tar xf phpMyAdmin-2.11.11.3-all-languages.tar.gz -C /var/www/
mv /var/www/phpMyadmin* /var/www/phpmyadmin && cd /var/www/phpmyadmin
cp config.sample.inc.php config.inc.php
vi config.inc.php 修改以下
$cfg['blowfish_secret'] = 'ADKFdkfjdl959435dfkds^%&';
```

安装daloradius管理freeradius

```
tar xf daloradius-0.9-9-rc1.tar.gz -C /var/www/
cd /var/www
mv daloradius* daloradius
cd /var/www/daloradius/contrib/db
Mysql -uroot -p radius < mysql-daloradius.sql
Mysql -uroot -p radius < fr2-mysql-daloradius-and-freeradius.sql
chown apache:apache /var/www/daloradius -R
chmod 644 /var/www/daloradius/library/daloradius.conf .php#配置文件需要给予写权限
```

vi /var/www/daloradius/library/daloradius.conf.php
```
$configValues['CONFIG_DB_ENGINE'] = 'mysql';
$configValues['CONFIG_DB_HOST'] = 'localhost';
$configValues['CONFIG_DB_USER'] = 'radiusadmin';
$configValues['CONFIG_DB_PASS'] = 'radiusadmin';
$configValues['CONFIG_DB_NAME'] = 'radius';
```

此处的radiusadmin账号是在phpmyadmin里创建的并给予radius库相应操作权限；

```
touch /tmp/daloradius.log
chown apache.apache /tmp/daloradius.log
```

vi /etc/httpd/conf/httpd.conf

```
Alias /radius "/var/www/daloradius/"
<Directory /var/www/daloradius/>
      Options None
      order deny,allow
      deny from all
      allow from 127.0.0.1
      allow from 192.168.0.0/24
</Directory>
Alias /phpmyadmin "/var/www/phpmyadmin/"
<Directory /var/www/phpmyadmin/>
      Options None
      order deny,allow
      deny from all
      allow from 127.0.0.1
      allow from 192.168.0.0/24
   </Directory>
```

启动apache  `/etc/init.d/httpd restart`

访问 http://192.168.0.2/radius

username: administrator, password: radius

在用户管理里面添加一个账号，即可用这个账号进行认证登陆。

windows客户端下载：http://swupdate.openvpn.net/community/releases/openvpn-2.2.0-install.exe

修改配置文件X:\Program Files\OpenVPN\config\client.ovpn
```
client
dev tap
proto udp
remote 192.168.0.2 1194
;remote-random
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
auth-user-pass
ns-cert-type server
;tls-auth ta.key 0
comp-lzo
verb 3
```

同时将服务器端的ca.crt拷贝到X:\Program Files\OpenVPN\config\  目录下。
win7需要以管理员身份运行OpenVPN GUI ，使用daloradius里添加的账号密码登陆。

添加ipv6支持：

让openvpn客户端支持Ipv6网络，需要将openvpn使用的虚拟网卡与具有ipv6地址的物理网卡进行桥接，让openvpn监听在此桥接的虚拟网卡上，然后通过自动DHCP实现为客户端分配ipv6地址。

安装配置网卡桥接：

```
yum  install bridge-utils
```

编辑bridge-start启动脚本，根据系统网络设备对应编辑。

下载radvd：http://www.litech.org/radvd/dist/radvd-1.8.tar.gz

```
tar  xf radvd-* && cd radvd-*
./configure --sysconfdir=/etc
make && make install
```

修改配置文件 vi /etc/radvd.conf

```
interface br0
{
    AdvSendAdvert on;
    MinRtrAdvInterval 3;
    MaxRtrAdvInterval 10;
    AdvDefaultPreference low;
    prefix 2001:250:1002:40::/64 //客户端IPv6前缀
    {
        AdvOnLink on;
        AdvAutonomous on;
        AdvRouterAddr on;
    };
};
```


参考文档：

[Authentication, Authorization & Accounting with FreeRadius & MySQL backend & web based Management with Daloradius](http://howtoforge.org/authentication-authorization-and-accounting-with-freeradius-and-mysql-backend-and-webbased-management-with-daloradius)
