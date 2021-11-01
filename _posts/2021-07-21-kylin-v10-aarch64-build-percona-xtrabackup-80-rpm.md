---
layout: post
title: "银行麒麟v10 aarch64机器构建percona-xtrabackup-80 rpm包"
description: "如何自己构建aarch64 xtrabackup rpm"
categories: [shell]
tags: [bash, ]
---


* Kramdown table of contents
{:toc .toc}

## 1 环境准备
```

yum install cmake3 openssl-devel libaio libaio-devel automake autoconf bison libtool ncurses-devel \
    libgcrypt-devel libev-devel libcurl-devel zlib-devel vim-common readline-devel python-sphinx rpm-build
```

## 2 获取最新SRPM包

```
# 查看需要下载的版本
https://repo.percona.com/yum/release/8/SRPMS/
#如：
wget https://repo.percona.com/yum/release/8/SRPMS/percona-xtrabackup-80-8.0.25-17.1.generic.src.rpm
```

## 3 BUILD RPM

```
rpm -ivh percona-xtrabackup-80-8.0.25-17.1.generic.src.rpm
cd ~/rpmbuild
rpmbuild -bb --nodebuginfo SPECS/percona-xtrabackup.spec
```
## OVER
