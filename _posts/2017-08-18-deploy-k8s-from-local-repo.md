---
layout: post
title: "k8s测试环境部署记录"
description: "记录k8s在本地测试环境的部署过程"
categories: [system]
tags: [k8s,kubernetes ]
---

> 本文记录了在本地部署k8s测试环境的过程，部署脚本参考github上其他同学分享的脚本，在其基础上做了些小改动。

* Kramdown table of contents
{:toc .toc}


## k8s 中的节点类型

master 负责管理其他节点的调度中心，master可以有备机replica做冗余

minion 由master管理，运行容器服务，1个集群中有N个minion节点

## 部署前准备

部署此测试环境参考了github上其他同学的分享，我fork的repo地址：https://github.com/5xops/k8s-deploy

首先按照github repo中的readme部分将k8s rpm包下载到本地，以准备离线部署。其次我在本地测试环境准备了6以上的虚拟机节点，其中3个节点用于部署etcd集群，2个节点用于部署k8s master，其余节点用于部署k8s minion。
所有虚拟机均运行在一台物理服务器上，管理虚拟机用了这个脚本（https://github.com/itxx00/vmm）。

首先创建好需要使用到的虚拟机节点：
```
vmm create etcd1
vmm create etcd2
vmm create etcd3
vmm create kubem1
vmm create kubem2
vmm create node1
vmm create node2
```

因部署k8s集群时要求所有节点都配置好主机名，因为默认创建出来的虚拟机没有修改hostname，需要使用另外一个脚本来配置hostname并配置好/etc/hosts，首先准备好初始化执行的脚本：
```
#!/bin/bash
# content of ~/.vmm/init.sh
cd /tmp
hostnm=$(cat hostname)
[[ -n $hostnm ]] || {
    echo "err"
    exit 1
}
echo $hostnm >/etc/hostname
hostname $hostnm
cat /tmp/hosts.tmp >/etc/hosts

```

接着执行初始化操作：
```
vminit etcd1
vminit etcd2
vminit etcd3
vminit kubem1
vminit kubem2
vminit node1
vminit node2
```
每个节点的hostname将被设置成虚拟机的名字，同时vminit脚本会把本地的ssh公钥和私钥都拷贝到虚拟机节点，这样初始化之后的节点可以通过相同的密钥互相免密登录，注意这样的操作仅适用于快速搭建测试环境，生产环境千万不要这么处理。打通免密ssh是因为后面部署k8s时会用到。

## 搭建etcd集群

在部署k8s集群之前，我们需要先部署一个独立的etcd集群，k8s会用到这个集群。这里我采用ansible playbook来部署，playbook已经分享到github上面，（https://github.com/itxx00/ansible-etcd），按照repo中的readme来准备好集群。

## 准备镜像仓库

将下载下来的k8s rpm包和docker镜像放到/data/k8s-deploy目录，其中rpms目录存放了需要用到的rpm包，images目录存放需要用到的docker镜像，因原脚本中不包含k8s dashboard（一套web ui管理界面），为了能够部署dashboard，对原脚本作了修改增加了部署dashboard的选项，部署dashboard需要一些额外的docker镜像和配置文件，这里先补充好docker镜像：
```
yum -y install docker
systemctl start docker
docker pull googlecontainer/kubernetes-dashboard-amd64:v1.6.1
docker pull googlecontainer/heapster-influxdb-amd64:v1.1.1
docker pull googlecontainer/heapster-grafana-amd64:v4.0.2
docker pull googlecontainer/heapster-amd64:v1.3.0
cd /data/k8s-deploly/images
docker save googlecontainer/kubernetes-dashboard-amd64 -o kubernetes-dashboard-amd64_v1.6.1.tar
docker save googlecontainer/heapster-influxdb-amd64 -o heapster-influxdb-amd64_v1.1.1.tar
docker save googlecontainer/heapster-grafana-amd64 -o heapster-grafana-amd64_v4.0.2.tar
docker save googlecontainer/heapster-amd64 -o heapster-amd64_v1.3.0.tar
```

原脚本中安装rpm包是在各节点下载好后本地安装的，我改进了一下采用yum方式安装，准备yum仓库：
```
createrepo /data/k8s-deploy/rpms
yum -y install nginx
cat >/etc/nginx/conf.d/k8srepo.conf <<EOF
server {
    listen       8000;
    server_name  _;
    root         /data/k8s-deploy;
    autoindex on;

    location / {
        autoindex on;
    }
}
EOF
systemctl restart nginx
```
至此yum repo和docker镜像准备完成。

## 部署k8s集群

部署过程参考https://github.com/5xops/k8s-deploy/blob/master/README.md ，需要修改repo中的k8slocal.repo文件中ip地址为yum仓库对应的ip地址，部署master、minion和replica可参考master.sh，minon.sh, replica.sh的内容。
需要注意的是为了部署dashboard服务我们额外增加了dashboard相关的配置文件，具体增加的内容请参考这个commit： https://github.com/5xops/k8s-deploy/commit/1766a675d76edb32f310acd98d5c6ed50a356e5b


至此，k8s测试环境及搭建完成，后续我将使用这个k8s测试环境来部署其他服务，后面会慢慢分享。

