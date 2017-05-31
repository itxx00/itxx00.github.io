---
layout: post
title: "为Bash增加命令执行日志"
description: "descripts a method for adding history logging on bash shell"
categories: [shell]
tags: [bash, hitory logging]
redirect_from:
  - /2011/02/17/
---

> 我们将修改bash的源码来实现”无敌”logging机制，也将看到”无敌”并不是真正的无敌。

* Kramdown table of contents
{:toc .toc}

## 一：bash为何需要logging
Bash堪称*nix世界使用最广泛的shell，其特性之一是历史命令(history)机制。此机制主要用于为用户提供方便－－少敲几下键盘，提高工作效率。然而，被广泛讨论的是bash_history可以用作logging机制以此来监控用户的活动。此文将对上述问题进行讨论并解释为啥logging机制在少数人面前会失效。我们将见到各种用于保护history文件的防御措施是如何不费吹灰之力或稍微费点力就被突破的。随着讨论的跟进，突破的限制也将变得更严，但这并不代表突破起来就更困难，与之相反大部分方法都是可以不费脑子的。最后，我们将修改bash的源码来实现”无敌”logging机制，也将看到”无敌”并不是真正的无敌。

## 二：加固bash_history

假设你所管理的系统提供shell登录功能，你的用户当中有个别及其讨人厌的家伙，于是你想监控他的活动，因为你非常怀疑他半夜三更使用你所负责保护的CPU和系统资源作恶意行为（或是其他的，例如下毛片等）。我们暂且叫他小明（此处原文为Bob，Bob一名在国外经常用来指代坏蛋）。

因为所有用户都是使用bash作为默认shell，你开始着手修改bash的配置文件：

### 第1步：使bash历史记录文件和相关文件无法被删除或修改。

小明所做的第一件事应该是建立history到/dev/null的链接。

~~~shell
bob$ rm ~/.bash_history
bob$ ln -s /dev/null ~/.bash_history
~~~

这可以通过修改历史记录文件为只能被追加来进行阻止，执行以下命令来改变其属性：

~~~
# chattr +a /home/bob/.bash_history
~~~

这是使用文件系统附加属性来指定文件只能被追加，大多数文件系统支持此功能(例如ext2/3,XFS,JFS)。在FreeBSD上可以执行：

~~~
# sappnd /home/bob/.bash_history
~~~

你还应修改shell启动相关的其他文件的这个属性：

~~~shell
# chattr +a /home/bob/.bash_profile
# chattr +a /home/bob/.bash_login
# chattr +a /home/bob/.profile
# chattr +a /home/bob/.bash_logout
# chattr +a /home/bob/.bashrc
~~~

前三个文件在交互式bash shell（或非交互式sehll使用–login选项）调用时被读取(在读完全局配置文件/etc/profile后)。.bashrc文件只在当non-login交互式shell调用时被读取。这意味着当小明已登进系统后，用以下方法自己调用一个新shell时:

`bob$ bash`

此时只有.bashrc文件被读取，而上面所列的前三个配置文件不会再次被读取了。

做了以上属性的修改后再来做更进一步的”加固”，一个所谓的保护措施。

### 第2步：配置 .bash*配置文件

所有的配置将针对.bashrc文件，因为其他三个配置文件本身会调用.bashrc，也就是说.bashrc无论如何都会被读取 (不管用户是否刚登录或是登录后手工调用bash shell)。

所以，所有修改都针对.bashrc的好处是可以防止小明登录后手工调用新的bash shell来跳过仅在.bash_profile,.bash_login,.profile三个配置文件中生效的配置选项，另一好处是这三个文件本身都会调用.bashrc，所以在首次登录系统时.bashrc当中的配置也会生效。

~~~shell
# cat >> /home/bob/.bashrc << EOF
> shopt -s histappend
> readonly PROMPT_COMMAND=”history -a”
> EOF
~~~

此处histappend选项的作用是让bash附加上最后一行$HISTSIZE给$HISTFILE文件（一般是~/.bash_history文件），不管交互式shell何时退出。默认的，bash每次均会覆盖$HISTFILE以保证只有一个session被保存以此来节约空间。

环境变量PROMPT_COMMAND会保存一条将被优先执行的命令，意思是说”history -a”命令将在用户执行命令前被优先执行，这将保证不管当前命令前一条是执行的什么，它将立即被追加进$HISTFILE，而不用等待整个会话结束再将历史命令从内存记录至硬盘。

此处的readonly作用是使变量不可修改以防止被小明覆盖掉或是直接屏蔽掉。

最后要完成的步骤是使所有与bash_history相关的环境变量都变为readonly:

~~~
readonly HISTFILE
readonly HISTFILESIZE
readonly HISTSIZE
readonly HISTCMD
readonly HISTCONTROL
readonly HISTIGNORE
~~~

### 第3步：禁掉系统中所有其他shell，一般包括csh,tcsh,ksh。

~~~
# chmod 750 csh
# chmod 750 tcsh
# chmod 750 ksh
~~~

这将阻止小明把bash shell切换成其他shell。

现在，机敏点的管理员会抱怨上面的都是shit！

还有一个shell逃出了我们的掌控！在你看完以上叙述跳入浮想联翩之前，让我们来搞清一些事情。

很久很久以前… （你懂的），原本只有一个Bourne shell 或者叫sh，现如今，/bin/sh实际上是/bin/bash的一个链接。Bash在被调用时检查它是以哪个名字被调用的并以此来判断是不是调用sh，它试图模仿历史版本的sh的行为并和POSIX标准保持一致。

如果以交互式login shell或非交互式shell带上–login选项启动，它才读取/etc/profile和~/.profile来初始化配置。如果以交互式shell被调用，则试图解释$ENV变量，当$ENV非空则使用它的值当作默认配置并执行。我们将在本文的下一节讨论如何利用这点来秒杀bash的所有设置。

## 三：攻破logging机制

现在是时候站在小明的角度来看下所有问题了。我们将验证上面的防御是如何一步步被攻破的。在实践中的可能性是无穷进的。

以下所提及的突破bash_history logging机制的技巧只是九牛一毛。

* 方法1：使用Bourne shell –/bin/sh逃脱术

`$ /bin/sh`

调用sh会导致bash模仿如前所述的历史版本sh而不会读取与bash直接相关的任何配置文件。因此，小明现在能够避开$HISTFILE变量了，

因为它已不再是readonly。

`$ unset HISTFILE`

这会使得logging机制在当前会话中直接萎掉，因为此变量控制的历史命令记录文件将会是空的。

注：也可以通过调用/bin/rbash（如果系统里存在的话）来实现相同效果，它会模仿受限版本的bash，和sh一样也是一个bash的链接，但是使用起来确实有些让人蛋疼。

* 方法2：让bash不加载.bashrc配置文件

可以通过以下方法实现：

`$ /bin/bash –norc`

这样即可禁止bash读取.bashrc从而被设置成readonly的变量变成了writeable，然后像下面这样做：

`$ HISTFILE=`

会清空$HISTFILE变量—>无历史记录。

## 四：Hacking bash-使用syslog日志接口

从以上我们很清楚地得出结论－－传统的加固bash_history的方法实际上都是扯淡。然而我们却可以更向前一步的hack bash本身来减少logging机制的脆弱性并提高其隐秘性。需要注意的是即便如此也是可以被攻破的。由于bash与内核的差距导致它并不是足够的健壮来作为一个logging设备，即便是hack了它的核心。

现在的想法是修改bash源码使用户键入的所有指令全部发送给syslog，由syslog将日志记录到/var/log目录下。我们将提供一个快速而且很黄很暴力的方法来实现这一目标－－这里，哪个用户键入的哪条指令将没有差别的被对待，而这也是可以被实现的。

我们的接口的最佳放置点是parse.y文件，它由bash的yacc语法组成。当一条指令在shell中被下达时bash解释器将迅速被调用。因此，将syslog钩子放置在解释器刚好完成它的工作前一点点，貌似是个好办法。需要修改的仅仅是增加两行代码：包含进syslog.h和设置syslog调用。我们使用了bash-3.2的源码：

~~~c
[ithilgore@fitz]$diff -E -b -c ~/bash-3.2/parse.y ~/hacked_bash/parse.y
*** ../../bash-3.2/bash-3.2/parse.y Tue Sep 19 13:37:21 2006
— parse.y Sat Jul 12 18:32:26 2008
***************
*** 19,24 ****
— 19,25 —-
Foundation, 59 Temple Place, Suite 330, Boston, MA 02111 USA. */
%{
 + #include #include “config.h”
 #include “bashtypes.h” 
 *************** *** 1979,1984 **** — 1980,1986 —
 - shell_input_line_len = i;
 /* == strlen (shell_input_line) */
 set_line_mbstate ();
 + syslog(LOG_LOCAL0 | LOG_CRIT, “%s”, shell_input_line);
 #if defined (HISTORY) if (remember_on_history && shell_input_line && shell_input_line[0])
~~~

上面的调用产生了一条日志消息，此消息将被syslog根据LOG_CRIT级别送到local0的设备上。要让这个东东生效则还必须要在/etc/syslog.conf配置文件中加入一条：

`local0.crit /var/log/hist.log`

至此用户下达的每条指令都将躺在/var/log/hist.log里，这个日志文件一般情况下日有root用户有读权限。

要注意的是上面所提到的hack并不区分是否为不同用户的输入。要实现的话还有更多的事情需要做的。由于所有的命令都被记录下来，那么由shell脚本执行或启动bash时的配置文件执行所产生的垃圾信息也是会被记录下来的。

现在唯一剩下的问题是”上面的hack要怎样才能被攻破？”其实这相当滴简单：

—->编译或上传一个你自己的干净的bash或其他shell即可搞定。

由于上面的hack是在特定版本的基础上的所以你编译或上传的干净bash可能在他的系统上会运行失败。

## 五：总结

Bash 只是一个shell，并不是一个logging设备，而bash_history只是用来为用户提供点方便少敲几下键盘而已。毫不装逼的说一句所有使用它来当监控设备的做法都是白搭。如果你是个较真的系统管理员且确实需要监控用户的活动，那就写个内核模块记录所有用户的键盘记录，并根据uid或其他参数进行过滤。这个方法将会非常管用并且很难被攻破（只是很难不是没那可能）。

现在已经有Linux包括FreeBSD下的审计框架可供选择。在FreeBSD平台，由Robert Watson和TrustedBSD项目开发的审计框架是选择之一。在linux平台，由来自红帽的Steve Grubb开发的Linux Auditing System也是一个选择：http://people.redhat.com/sgrubb/audit/

## 六：参考资料
* a. bash & syslog man pages

* b. bash-3.2 source code -http://ftp.gnu.org/gnu/bash/bash-3.2.tar.gz

