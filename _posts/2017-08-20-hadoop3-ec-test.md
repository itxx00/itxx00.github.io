---
layout: post
title: "HADOOP3.0.0纠删码测试"
description: "描述"
categories: [bigdata]
tags: [hadoop,bigdata]
---

> 记录hadoop3.0.0版本测试安装过程，对hadoop3.0.0中纠删码进行了简单测试。

* Kramdown table of contents
{:toc .toc}

## 基础环境

HADOOP3.0.0版本中增加了纠删码技术，在提高可用性的同时还能减低存储成本，目前处于实验阶段，以下将测试环境中的搭建步骤及简单测试过程进行记录。本次仅对hdfs进行测试，因此不会部署其他服务。

系统环境如下：

kvm虚拟机，1个namenode节点，6个datanode节点，4core ，8G mem ， 50G disk

hadoop版本：3.0.0-alpha4

java版本： 1.8.0_144

## 测试集群安装

### 基础包安装

从apache镜像下载hadoop-3.0.0-alpha4，当前最新版本，hadoop home目录/opt/hadoop，在namenode节点将配置等修改好之后拷贝到所有datanode节点。

~~~ bash
tar xf hadoop-3.0.0-alpha4.tar.gz
mv hadoop-3.0.0-alpha4 /opt/hadoop
yum -y install jdk --disablerepo=* --enablerepo=local-custom
~~~

### 配置文件修改

需要修改的配置文件有hadoop-env.sh 、core-site.xml、hdfs-site.xml


修改`/opt/hadoop/etc/hadoop/hadoop-env.sh`中以下参数：

~~~ bash
export JAVA_HOME=/usr/java/jdk1.8.0_144
export HADOOP_HOME=/opt/hadoop
# 注意这里配置heapsize和2.7版本的差别，2.7为HADOOP_HEAPSIZE
export HADOOP_HEAPSIZE_MAX=1024
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true”
export HADOOP_CLIENT_OPTS="-Xmx512m $HADOOP_CLIENT_OPTS”
export HADOOP_LOG_DIR=${HADOOP_HOME}/logs
export HADOOP_PID_DIR=/tmp
~~~

修改`/opt/hadoop/etc/hadoop/hdfs-site.xml`内容如下：

~~~ xml
<configuration>
   <property>
      <name>dfs.blocksize</name>
      <value>134217728</value>
    </property>
   <property>
      <name>dfs.datanode.data.dir</name>
      <value>/data/hdfs/data</value>
      <final>true</final>
    </property>
    <property>
      <name>dfs.namenode.name.dir</name>
      <value>/data/hdfs/namenode</value>
      <final>true</final>
    </property>
   <property>
      <name>dfs.namenode.rpc-address</name>
      <value>192.168.199.26:8020</value>
    </property>
</configuration>
~~~

将环境变量增加到当前用户bashrc：

~~~ bash
# ~/.bashrc
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:/opt/hadoop/bin
~~~

配置修改完成后将/opt/hadoop整个目录同步到所有datanode节点。

## 启动HDFS

### 格式化namenode

~~~ bash
hdfs namenode -format ns1
~~~

### 启动namenode和datanode

~~~ bash
# 注意3.0.0版本的差别，在2.7中启动脚本如下
hadoop-daemon.sh --config $HADOOP_HOME/etc/hadoop --script hdfs start namenode
hadoop-daemon.sh --config $HADOOP_HOME/etc/hadoop --script hdfs start datanode
# 新版本中已经重写了管理脚本，统一到hdfs命令中，启动方式如下：
hdfs --daemon start namenode
hdfs --daemon start datanode
~~~

启动成功之后即可通过[namenode web ui](http://192.168.199.26:9870/dfshealth.html#tab-datanode)观察到集群基本情况，2.7中默认web ui端口为50070，而3.0.0中修改为9870，

## 测试hdfs
启动完成之后可以通过hdfs命令测试服务是否可用：

~~~ bash
hdfs dfs -mkdir hdfs://192.168.199.26:8020/t1/
dd if=/dev/urandom of=f1 bs=1M count=5000
hdfs dfs -put f1 hdfs://192.168.199.26:8020/t1/
hdfs dfs -rm -skipTrash hdfs://192.168.199.26:8020/t1/f1
~~~

这里使用的时候需要写完整的hdfs协议和namenode:port，我们可以修改一下配置文件，将defaultfs修改为hdfs协议，方便测试。同时此版本中默认并未启用纠删码，需要手工配置。
默认内置支持的policy有 RS-3-2-64k, RS-6-3-64k, RS-10-4-64k, RS-LEGACY-6-3-64k, XOR-2-1-64k，我这里只准备了少量节点，因此只对其中两种进行简单测试。

修改`hdfs-site.xml` 增加以下配置：

~~~ xml
    <property>
      <name>dfs.namenode.ec.policies.enabled</name>
      <value>XOR-2-1-64k,RS-3-2-64k</value>
    </property>
    <property>
      <name>dfs.nameservices</name>
      <value>ns1</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.ns1</name>
        <value>nn1</value>
    </property>
    <property>
      <name>dfs.namenode.rpc-address.ns1.nn1</name>
      <value>192.168.199.26:8020</value>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.ns1</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
~~~

修改`core-site.xml`增加以下配置：

~~~ xml
<configuration>
    <property>
      <name>fs.defaultFS</name>
      <value>hdfs://ns1</value>
      <final>true</final>
    </property>
</configuration>
~~~

重启namenode节点后，就可以使用纠删码了，纠删码可以针对目录进行设置，不同的目录设置不同的策略。

~~~ bash
hdfs ec -setPolicy -policy XOR-2-1-64k -path /t1
hdfs ec -setPolicy -policy RS-3-2-64k -path /t2
~~~

我们准备了3个目录，t1和t2分别设置了不同的policy，t3不设置，往3个目录上传相同的5G大小的文件，每次上传前均清空hdfs中数据，并清空虚拟机和物理机缓存，统计的耗时和空间占用情况大致如下：

|-----------------+------------+-----------------+----------------|
| HDFS目录        | 纠删码策略 |     put耗时     | 磁盘占用kb     |
|-----------------|:-----------|:---------------:|---------------:|
| t1              |XOR-2-1-64k | 1m15.559s       | 7740364        |
| t2              |RS-3-2-64k  | 1m13.920s       | 8600436        |
| t3              |无(三副本)  | 2m7.705s        | 15480600       |
|-----------------+------------+-----------------+----------------|

通过简单的测试对比，我们可以大致了解到在理想状态下纠删码技术比传统的三副本在写入速度上有提升，因其降低了对磁盘IO和带宽的消耗，同时占用的磁盘空间小于三副本方式。其中磁盘占用和不同的纠删码策略理论值基本吻合， 磁盘空间消耗倍数为 `(校验块+数据块)/数据块`。

注意这里的测试仅限于对HADOOP3.0.0中纠删码有一个感性认识，测试方法有很多不严谨的地方，比如put数据的耗时并未考虑各种环境因素，仅仅是在一个相对理想的环境下进行简单测试。实际生产环境中情况非常复杂，需要权衡CPU带宽磁盘，甚至机架供电等各方面因素，如果需要获得一份可靠的性能对比数据则必须保障稳定运行足够长的时间，通过长期观察才能得出对生产有实际指导意义的信息。


参考文档：

[1] http://hadoop.apache.org/docs/r3.0.0-alpha4/hadoop-project-dist/hadoop-common/ClusterSetup.html

[2] http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html
