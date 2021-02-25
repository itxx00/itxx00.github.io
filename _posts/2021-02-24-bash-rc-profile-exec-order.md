---
layout: post
title: "bashrc与profile的加载顺序"
description: "看下bashrc和profile的执行顺序到底是什么样的"
categories: [shell]
tags: [bash, ]
---

> 在使用bashrc和profile设置环境变量时，如果多个地方都有同一个变量的设置，则需要注意不同配置文件的加载顺序问题

* Kramdown table of contents
{:toc .toc}

## 背景
如果加载顺序没弄明白，有可能会在使用过程中遇到各种困扰，比如为什么设置了profile但是环境变量不生效？为什么变量ssh后获取的不一样？下面我们以CentOS7系统为例，通过一个简单的小实验来观察下到底bash的几个配置文件加载顺序是怎样的。

我们知道可以用来设置环境变量的文件常用的有以下几个：
- /etc/profile
- /etc/profile.d/*.sh
- /etc/bashrc
- ~/.bash_profile
- ~/.bashrc


而不同的文件加载时机又分为login shell和non-login shell两种情况。这两种情况需要区分对待，及不同的文件要在对应场景下才能生效。假设有一个相同的变量设置出现在各个文件里面，通过对不同文件的变量值进行差异设置即可观察出各个配置的加载优先级和生效情况。

## 实验
先写入各个配置文件如下：
```
# tail -n1 /etc/profile /etc/bashrc /etc/profile.d/well.sh ~/.bash_profile ~/.bashrc
==> /etc/profile <==
export WELL=etc-profile

==> /etc/bashrc <==
export WELL=etc-bashrc

==> /etc/profile.d/well.sh <==
export WELL=etc-profile-d

==> /root/.bash_profile <==
export WELL=home-bash-profile

==> /root/.bashrc <==
export WELL=home-bashrc
```

接下来开始观察，需要注意的是每次修改配置之后新开shell重新加载环境配置：

```
[root@localhost ~]# echo $WELL
home-bash-profile
[root@localhost ~]# ssh localhost 'echo $WELL'
home-bashrc
[root@localhost ~]#


[root@localhost ~]# sed -i '$d' ~/.bashrc
[root@localhost ~]# sed -i '$d' ~/.bash_profile
[root@localhost ~]#


[root@localhost ~]# echo $WELL
etc-bashrc
[root@localhost ~]# ssh localhost 'echo $WELL'
etc-bashrc
[root@localhost ~]#


[root@localhost ~]# sed -i '$d' /etc/bashrc


[root@localhost ~]# echo $WELL
etc-profile
[root@localhost ~]# ssh localhost 'echo $WELL'
etc-profile-d
[root@localhost ~]#

# 重新写入~/.bashrc后
[root@localhost ~]# echo $WELL
home-bashrc
[root@localhost ~]# ssh localhost 'echo $WELL'
etc-profile-d
[root@localhost ~]#


# 重新写入~/.bash_profile,去掉~/.bashrc后
[root@localhost ~]# echo $WELL
home-bash-profile
[root@localhost ~]# ssh localhost 'echo $WELL'
etc-profile-d
[root@localhost ~]#

```

需要注意的是以上测试是将变量放到每个配置末行，因为配置之间有互相加载的机制，如果放在其他位置则测试结果会不一样。

## 结论
观察上面的结果，可以得出以下实验结论：

1 login shell会加载所有配置,优先级为~/.bash_profile ~/.bashrc /etc/bashrc /etc/profile /etc/profile.d

2 non-login shell时加载优先级为 ~/.bashrc /etc/bashrc /etc/profile.d

3 non-login shell不会加载的配置有 ~/.bash_profile /etc/profile

4 两种情况下都会加载的有~/.bashrc /etc/bashrc /etc/profile.d

那么如果我们需要在系统全局设置一个环境变量，要保证login shell和non-login shell都能表现一致，需要如何设置呢？

因为~/.bashrc为用户局部配置文件，不影响全局，而/etc/bashrc为系统内置文件不建议修改，如果是有全局环境变量需要设置建议放置到/etc/profile.d


over.
