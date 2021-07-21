---
layout: post
title: "UOS arm64机器build percona-xtrabackup-80 deb包"
description: "uos如何快速构建xtrabackup deb"
categories: [shell]
tags: [bash, ]
---


* Kramdown table of contents
{:toc .toc}

## 1 系统环境

```
root@VM-0-14-linux:~# cat /etc/os-release
PRETTY_NAME="uos 20"
NAME="uos"
VERSION_ID="20"
VERSION="20"
ID=uos
HOME_URL="https://www.chinauos.com/"
BUG_REPORT_URL="http://bbs.chinauos.com"

root@VM-0-14-linux:~# uname -a
Linux VM-0-14-linux 4.19.0-arm64-server #1635 SMP Mon Jan 13 16:07:12 CST 2020 aarch64 GNU/Linux

root@VM-0-14-linux:~# cat /etc/debian_version
10.1
root@VM-0-14-linux:~#

```

## 2 配置perconca官方apt源

```
wget https://repo.percona.com/apt/percona-release_latest.buster_all.deb
dpkg -i percona-release_latest.buster_all.deb
# 修改脚本中两个变量
vi /usr/bin/percona-release
CODENAME=buster
OS_VER=buster
# 开启perconca源
percona-release enable-only tools release
```

## 3 BUILD
```
# 安装依赖
apt-get build-dep percona-xtrabackup-80
# 构建
apt-get source --compile percona-xtrabackup-80
```

## OVER
