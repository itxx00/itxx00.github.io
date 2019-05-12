---
layout: post
title: "很诡异的服务日志无法切割问题分析"
description: "遇到某服务进程产生的日志始终无法切割，原来原因在这里"
categories: [system]
tags: [system,shell]
---

> 遇到某服务进程产生的日志始终无法切割，原来原因在这里

* Kramdown table of contents
{:toc .toc}

## 问题现象

某业务机器因磁盘容量超过阈值收到告警，分析发现是由于该机器上某个服务进程产生的日志文件量太大，且日志文件未按周期切割，进而导致历史日志信息积累到单个日志文件中。
未避免故障发生采取临时措施手工切割该日志文件，由于该服务进程并未提供内置的日志切割，因此手工模拟类似logrotate的```copytruncate```模式对日志进行切割。
但在将日志truncate之后，奇怪的一幕发生了，通过ls查看文件大小发现并未减少，直觉判断这可能是文件句柄一直处于打开状态且偏移量未发生改变导致。
在进一步检查了该进程的启动方式之后，发现该进程通过nohup启动，并将标准输出重定向到持续增大的日志文件中。

## 模拟

我们通过下面几行脚本来模拟此现象：

~~~shell
#!/bin/bash
while true; do
    sleep 1
    head -5000 /dev/urandom
done
~~~

脚本启动后会有一个常驻进程每个1秒钟输出一堆字符串以此来模拟日志文件增涨，我们按照以下方式启动：

~~~shell
nohup ./daemon.sh >out.log 2>&1 < /dev/null &
~~~

等待一会之后我们观察到日志已经写入了

~~~
[root@localhost t]# ll -h out.log ;du -h out.log 
-rw-r--r-- 1 root root 64M Oct 19 17:41 out.log
64M   out.log
~~~

接着将日志文件清空，再观察文件大小变化

~~~
[root@localhost t]# 
[root@localhost t]# truncate -s0 out.log              
[root@localhost t]# ll -h out.log ;du -h out.log 
-rw-r--r-- 1 root root 93M Oct 19 17:41 out.log
4.0M  out.log
~~~

这时可以看到，虽然文件被清空了，但是ls看到的大小依然没有发生变化，也就是说文件中产生了大量空洞。


## 解决方法

将nohup启动进程后的输出重定向 ```>``` 替换为 ```>>```， 即改为append模式来写入日志，这时再truncate就不会出现上面的问题了。

~~~
 nohup ./daemon.sh >>out.log 2>&1  </dev/null &
 
 
[root@localhost t]# ll -h out.log ;du -h out.log 
-rw-r--r-- 1 root root 48M Oct 19 19:43 out.log
64M   out.log
[root@localhost t]# ll -h out.log ;du -h out.log 
-rw-r--r-- 1 root root 77M Oct 19 19:43 out.log
128M  out.log
[root@localhost t]# truncate -s0 out.log              
[root@localhost t]# ll -h out.log ;du -h out.log 
-rw-r--r-- 1 root root 1.3M Oct 19 19:43 out.log
2.0M  out.log
~~~

这里留一个问题： 为什么使用```append```模式就不会出现这个问题？

参考文档：

[1] https://www.gnu.org/software/bash/manual/bash.html#Redirections

[2] https://www.gnu.org/software/coreutils/manual/html_node/nohup-invocation.html
