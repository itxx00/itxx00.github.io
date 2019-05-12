---
layout: post
title: "CentOS7系统安装ansible awx记录"
description: "记录awx的安装测试过程以及需要注意的点"
categories: [system]
tags: [system,shell]
---

> 记录awx的安装测试过程以及需要注意的点

* Kramdown table of contents
{:toc .toc}

## 关于awx

awx(https://github.com/ansible/awx)是ansible tower的开源版本，作为tower的upstream对外开源，
项目从2013年开始维护，2017年由redhat对外开源，目前维护得比较活跃。由于官方的install guide写得有点杂不是很直观，
导致想要安装个简单的测试环境体验一下功能都要折腾半天，这里提供一个简单版本的安装流程方便快速体验。

## 安装过程

安装软件包

```
yum -y install epel-release
systemctl disable firewalld
systemctl stop firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
yum -y install git gettext ansible docker nodejs npm gcc-c++ bzip2 python-docker-py
```
启动服务

```
systemctl start docker
systemctl enable docker
```

clone awx代码

```
git clone https://github.com/ansible/awx.git
cd awx/installer/
# 注意修改一下postgres_data_dir到其他目录比如/data/pgdocker
vi inventory
ansible-playbook -i inventory install.yml
```

检查日志

```
docker logs -f awx_task
```

以上是安装过程，因为本地环境访问外网要经过代理，这里记录一下配置docker通过代理访问外网的方式，否则pull image会有问题。

```
mkdir /etc/systemd/system/docker.service.d/
cat > /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=proxy.test.dev:8080" "HTTPS_PROXY=proxy.test.dev:8080" "NO_PROXY=localhost,127.0.0.1,172.1.0.2"
EOF

systemctl daemon-reload
systemctl restart docker
systemctl show --property=Environment docker
```




参考文档：

[1] http://khmel.org/?p=1245

[2] https://docs.docker.com/engine/admin/systemd/#httphttps-proxy
