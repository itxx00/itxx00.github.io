---
layout: post
title: "浅析snapshots, blockcommit,blockpull"
description: "show how to create snapshot with libvirt and qemu-img."
categories: [virt]
tags: [libvirt, kvm, snapshot]
redirect_from:
  - /2012/11/23/
---

> 这是一篇关于snapshots, blockpull, blockcommit的的介绍.作者和with Eric Blake, Jeff Cody,Kevin Wolf以及很多IRC和mailing lists里面的同学大量讨论以及作者大量的特向测试的基础之上总结出来的

* Kramdown table of contents
{:toc .toc}

## 基础知识

一个虚拟机快照可被看作是虚拟机的在某个指定时间的视图（包括他的操作系统和所有的程序）.据此，某可以还原到一个之前的完整的状态，或者在guest运行的时候做个备份.所以，在我们继续深入之前我们必须搞懂两个名词：backing files和overlays .

### QCOW2 backing files 与 overlays

qcow2（qemu copy-on-write）具有创建一个base-image，以及在base-image（即backing file）的基础上创建多个copy-on-write overlays镜像的能力.backing files和overlays十分有用，可以迅速的创建瘦装备虚拟机的实例，特别是在开发测试的时候可以让你迅速的回滚到之前的某个已知状态，丢弃overlay.

Figure-1

```
.--------------.    .-------------.    .-------------.    .-------------.
|              |    |             |    |             |    |             |
| RootBase     |<---| Overlay-1   |<---| Overlay-1A  <--- | Overlay-1B  |
| (raw/qcow2)  |    | (qcow2)     |    | (qcow2)     |    | (qcow2)     |
'--------------'    '-------------'    '-------------'    '-------------'
```

上图表明rootbase是overlay-1的backing file，以此类推.

Figure-2

```
.-----------.   .-----------.   .------------.  .------------.  .------------.
|           |   |           |   |            |  |            |  |            |
| RootBase  |<--- Overlay-1 |<--- Overlay-1A <--- Overlay-1B <--- Overlay-1C |
|           |   |           |   |            |  |            |  | (Active)   |
'-----------'   '-----------'   '------------'  '------------'  '------------'
   ^    ^
   |    |
   |    |       .-----------.    .------------.
   |    |       |           |    |            |
   |    '-------| Overlay-2 |<---| Overlay-2A |
   |            |           |    | (Active)   |
   |            '-----------'    '------------'
   |
   |
   |            .-----------.    .------------.
   |            |           |    |            |
   '------------| Overlay-3 |<---| Overlay-3A |
                |           |    | (Active)   |
                '-----------'    '------------'
```

上图表明我们可以只用单个backing file来创建多条链.

注意 : backing file 总是 只读 打开的. 换言之, 一旦新快照被创建，他的后端文件就不能被修改,(快照依赖于后端文件的这种状态).了解更多参见后面的('blockcommit' 节) .

示例 :

```
[FedoraBase.img] ----- <- [Fed-guest-1.qcow2] <- [Fed-w-updates.qcow2] <- [Fedora-guest-with-updates-1A]
                 \
                  \--- <- [Fed-guest-2.qcow2] <- [Fed-w-updates.qcow2] <- [Fedora-guest-with-updates-2A]
```

（注意箭头的方向，Fed-w-updates.qcow2 的backing file是 Fed-guest-1.qcow2）

上面的示例中可以看到 FedoraBase.img 安装了一个fedora17系统，并作为我们的backing file.现在这个backing file将作为模板快速的创建两个瘦装备实例，和 Figure-2 道理是一样的.

## 具体操作
使用qemu-img为单个backing file来创建两个fedora的瘦装备克隆:

```
# qemu-img create -b /export/vmimages/RootBase.img -f qcow2 \
  /export/vmimages/Fedora-guest-1.qcow2

# qemu-img create -b /export/vmimages/RootBase.img -f qcow2 \
  /export/vmimages/Fedora-guest-2.qcow2
```

现在，上面创建出来的两个镜像 Fedora-guest-1 & Fedora-guest-2 都可以用来启动一个虚拟机，继续我们的示例，现在我们需要创建一个f17的实例，但是这次我们需要创建的是具有完整的更新的实例，这时可以创建另外一个overlay（Fedora-guest-with-updates-1A）而这个overlay的backing file是'Fed-w-updates.qcow2'（一个包含了完整更新的镜像）:

```
# qemu-img create -b /export/vmimages/Fed-w-updates.qcow2 -f qcow2 \
   /export/vmimages/Fedora-guest-with-updates-1A.qcow2
```

我们可以使用qemu-img命令来查看镜像的信息，包括虚拟磁盘大小，使用大小，backing file指向:

```
# qemu-img info /export/vmimages/Fedora-guest-with-updates-1A.qcow2
```

注意: 最新版本的qemu-img可以递归查询到整条完整的链:

```
# qemu-img info --backing-chain /export/vmimages/Fedora-guest-with-updates-1A.qcow2
```

名词解释Snapshot:

* 内置快照（Internal Snapshots） -- 单个qcow2镜像文件存储了包括数据以及快照的状态信息，

内置快照又可以细分一下:-

  * 内置磁盘快照（Internal disk snapshot）:

  * 快照点的磁盘状态，数据和快照保存在单个qcow2文件中，虚拟机运行状态和关闭状态都可以创建.

Libvirt 使用 'qemu-img' 命令创建关机状态的磁盘快照.Libvirt 使用 'savevm' 命令创建运行状态的磁盘快照.

* 内置系统还原点（Internal system checkpoint）:

内存状态，设备状态和磁盘状态，可以为运行中的虚拟机创建，所有信息都存储在同一个qcow2文件中，只有在运行状态才能创建内置系统还原点.

Libvirt 使用'savevm' 命令来创建这种快照

* 外置快照（External Snapshots） -- 当一个快照被创建时，创建时当前的状态保存在当前使用的磁盘文件中，即成为一个backing file.此时一个新的overlay被创建出来保存往后的数据.

这个也可以细分一下:-

  * 外置磁盘快照（External disk snapshot）: 磁盘的快照被保存在一个文件中，创建时间点以后的数据被记录到一个新的qcow2文件中.同样可以在运行和关闭状态创建.

Libvirt 使用 'transaction' 命令来为运行状态创建这种快照.
Libvirt 使用'qemu-img' 命令为关闭状态创建这种快照(截止目前功能还在开发中).
  * 外置系统还原点（External system checkpoint）:

虚拟机的磁盘状态将被保存到一个文件中，内存和设备的状态将被保存到另外一个新的文件中，

（这个功能也还在开发中）.

VM状态（VM state）:

保存运行状态虚拟机的内存设备状态信息至文件，可以通过此文件恢复到保存时的状态，有点类似系统的休眠.（注意创建VM状态保存的时候VM磁盘必须是未发生写入改动的）

Libvirt使用 'migrate' (to file)命令来完成VM状态转储.

## 创建snapshots

每次产生一个外置snapshot，一个 /new/ overlay 镜像就会随之生成，而前一个镜像就变成了一个快照.

diskonly内置快照创建

假如需要为名为'f17vm1'的虚拟机创建一个运行态或关闭态的内置快照snap1

```
# virsh snapshot-create-as f17vm1  snap1 snap1-desc
```
列出快照列表，使用*qemu-img*查看info

```
# virsh snapshot-list f17vm1
# qemu-img info /home/kashyap/vmimages/f17vm1.qcow2
```

disk-only外置快照创建 :

查看虚拟机磁盘列表

```
# virsh domblklist f17-base
Target     Source
---------------------------------------------
vda        /export/vmimages/f17-base.qcow2

```
创建外置disk-only磁盘快照（VM*运行态*）:

```
# virsh snapshot-create-as --domain f17-base snap1 snap1-desc \
--disk-only --diskspec vda,snapshot=external,file=/export/vmimages/sn1-of-f17-base.qcow2 \
--atomic
Domain snapshot snap1 created
```

    * 一旦上面的命令被执行，则原来的镜像f17-base将变为backing file，一个新的镜像被创建.

现在再列表查看虚拟机磁盘，你会发现新产生的镜像已经投入使用.

```
# virsh domblklist f17-base
Target     Source
----------------------------------------------------
vda        /export/vmimages/sn1-of-f17-base.qcow2

```

快照回滚

截止写此文之时，回滚至'内置快照'(system checkpoint或disk-only)是可以使用的.

虚拟机f17vm1回滚至快照'snap1'

```
# virsh snapshot-revert --domain f17vm1 snap1
```

使用 snapshot-revert 回滚 '外置磁盘快照' 稍微复杂些，需要涉及到稍微复杂点的问题，需要考虑的是合并'base'至'top'还是合并'top'至'base'.也就是说，有两种方式可以选择，外置磁盘快照链的缩短可以使用 blockpull 或 blockcommit .截止目前上游社区仍然在努力完善这项功能.

## 合并快照文件

外置快照非常有用，但这里有一个问题就是如何合并快照文件来缩短链的长度，如上所述这里

有两种方式:

* blockcommit: 从 top 合并数据到 base (即合并overlays至backing files).
* blockpull: 将backing file数据合并至overlay中.从 base 到 top .

### blockcommit

blockcommit可以让你将'top'镜像(在同一条backing file链中)合并至底层的'base'镜像.一旦 blockcommit 执行完成，处于最上面的overlay链关系将被指向到底层的overlay或base.这在创建了很长一条链之后用来缩短链长度的时候十分有用.

下面来个图说明下:

我们现在有一个镜像叫'RootBase'，拥有4个外置快照，'Active'为当前VM写入数据的，

使用'blockcommit'可以有以下多种case :

合并Snap-1, Snap-2 and Snap-3 至 'RootBase'
只合并Snap-1 and Snap-2 至 RootBase
只合并Snap-1 至 RootBase
合并Snap-2 至 Snap-1
合并Snap-3 至 Snap-2
合并Snap-2 和 Snap-3 至 Snap-1
注: 合并'Active'层(最顶部的overlay)至backing_files的功能还在开发中.

(下图解释case (6))

Figure-3

```
.------------.  .------------.  .------------.  .------------.  .------------.
|            |  |            |  |            |  |            |  |            |
| RootBase   <---  Snap-1    <---  Snap-2    <---  Snap-3    <---  Snap-4    |
|            |  |            |  |            |  |            |  | (Active)   |
'------------'  '------------'  '------------'  '------------'  '------------'
                                 /                  |
                                /                   |
                               /  commit data       |
                              /                     |
                             /                      |
                            /                       |
                           v           commit data  |
.------------.  .------------. <--------------------'           .------------.
|            |  |            |                                  |            |
| RootBase   <---  Snap-1    |<---------------------------------|  Snap-4    |
|            |  |            |       Backing File               | (Active)   |
'------------'  '------------'                                  '------------'
```

举个例子，有以下场景：

当前: [base] <- sn1 <- sn2 <- sn3 <- sn4(this is active)

目标: [base] <- sn1 <- sn4 (如此来丢弃sn2,sn3)

  下面有两种方式，method-a更快,method-b 慢些，但是sn2有效可用. (VM运行态).

            (method-a):

```
           # virsh blockcommit --domain f17 vda --base /export/vmimages/sn1.qcow2  \

               --top /export/vmimages/sn3.qcow2 --wait --verbose
```
[OR]
            (method-b):
```
# virsh blockcommit --domain f17 vda  --base /export/vmimages/sn2.qcow2  \
    --top /export/vmimages/sn3.qcow2 --wait --verbose
# virsh blockcommit --domain f17 vda  --base /export/vmimages/sn1.qcow2  \
    --top /export/vmimages/sn2.qcow2 --wait --verbose
```

注: 如果手工执行*qemu-img*命令完成的话, 现在还只能用method-b.
Figure-4

```
.------------.  .------------.  .------------.  .------------.  .------------.
|            |  |            |  |            |  |            |  |            |
| RootBase   <---  Snap-1    <---  Snap-2    <---  Snap-3    <---  Snap-4    |
|            |  |            |  |            |  |            |  | (Active)   |
'------------'  '------------'  '------------'  '------------'  '------------'
                  /                  |             |
                 /                   |             |
                /                    |             |
   commit data /         commit data |             |
              /                      |             |
             /                       | commit data |
            v                        |             |
.------------.<----------------------|-------------'            .------------.
|            |<----------------------'                          |            |
| RootBase   |                                                  |  Snap-4    |
|            |<-------------------------------------------------| (Active)   |
'------------'                  Backing File                    '------------'
```

上图演示了case1的blockcommit走向，现在sn4的backing file指向rootbase.

### blockpull

blockpull（qemu中也称作'block stream'）可以将backing合并至active，与blockcommit正好相反.截止目前只能将backing file合并至当前使用的active中，也就是说还不支持指定top的合并.
设想一个下面的场景:

Figure-5

```
.------------.  .------------.  .------------.  .------------.  .------------.
|            |  |            |  |            |  |            |  |            |
| RootBase   <---  Snap-1    <---  Snap-2    <---  Snap-3    <---  Snap-4    |
|            |  |            |  |            |  |            |  | (Active)   |
'------------'  '------------'  '------------'  '------------'  '------------'
                         |                 |              \
                         |                 |               \
                         |                 |                \
                         |                 |                 \ stream data
                         |                 | stream data      \
                         | stream data     |                   \
                         |                 |                    v
     .------------.      |                 '--------------->  .------------.
     |            |      '--------------------------------->  |            |
     | RootBase   |                                           |  Snap-4    |
     |            | <---------------------------------------- | (Active)   |
     '------------'                 Backing File              '------------'
```

使用blockpull我们可以将snap-1/2/3中的数据合并至active层，最终rootbase将变成active的直接后端.

命令如下:

假设快照已经使用 创建Snapshots 小节中的方式完成:

如*Figure-5*中描述的-- [RootBase] <- [Active].

```
# virsh blockpull --domain RootBase  \
  --path /var/lib/libvirt/images/active.qcow2  \
  --base /var/lib/libvirt/images/RootBase.qcow2  \
  --wait --verbose
```

后续的工作是我们需要使用virsh来清理掉不用的快照

```
# virsh snapshot-delete --domain RootBase Snap-3 --metadata
# virsh snapshot-delete --domain RootBase Snap-2 --metadata
# virsh snapshot-delete --domain RootBase Snap-1 --metadata
```

Figure-6

```
.------------.  .------------.  .------------.  .------------.  .------------.
|            |  |            |  |            |  |            |  |            |
| RootBase   <---  Snap-1    <---  Snap-2    <---  Snap-3    <---  Snap-4    |
|            |  |            |  |            |  |            |  | (Active)   |
'------------'  '------------'  '------------'  '------------'  '------------'
      |                  |              |                  \
      |                  |              |                   \
      |                  |              |                    \  stream data
      |                  |              | stream data         \
      |                  |              |                      \
      |                  | stream data  |                       \
      |  stream data     |              '------------------>     v
      |                  |                                    .--------------.
      |                  '--------------------------------->  |              |
      |                                                       |  Snap-4      |
      '---------------------------------------------------->  | (Active)     |
                                                              '--------------'
                                                                'Standalone'
                                                                (w/o backing
                                                                file)
```

上图表示的是将所有backing file全部合并至active

如下执行命令:

(1) 在我们执行合并 *之前* 查看一下快照的大小(注意观察'Active'):
    ::
```
        # ls -lash /var/lib/libvirt/images/RootBase.img
        608M -rw-r--r--. 1 qemu qemu 1.0G Oct 11 17:54 /var/lib/libvirt/images/RootBase.img

        # ls -lash /var/lib/libvirt/images/*Snap*
        840K -rw-------. 1 qemu qemu 896K Oct 11 17:56 /var/lib/libvirt/images/Snap-1.qcow2
        392K -rw-------. 1 qemu qemu 448K Oct 11 17:56 /var/lib/libvirt/images/Snap-2.qcow2
        456K -rw-------. 1 qemu qemu 512K Oct 11 17:56 /var/lib/libvirt/images/Snap-3.qcow2
        2.9M -rw-------. 1 qemu qemu 3.0M Oct 11 18:10 /var/lib/libvirt/images/Active.qcow2
```
(2) 单独检查下 'Active' 所指向的backing file ::
```
        # qemu-img info /var/lib/libvirt/images/Active.qcow2
        image: /var/lib/libvirt/images/Active.qcow2
        file format: qcow2
        virtual size: 1.0G (1073741824 bytes)
        disk size: 2.9M
        cluster_size: 65536
        backing file: /var/lib/libvirt/images/Snap-3.qcow2
```
(3) 开始 **blockpull** 操作.
    ::
```
        # virsh blockpull --domain ptest2-base --path /var/lib/libvirt/images/Active.qcow2 --wait --verbose
        Block Pull: [100 %]
        Pull complete
```
(4) 再检查下快照大小， 'Active'变得很大
    ::
```
        # ls -lash /var/lib/libvirt/images/*Snap*
         840K -rw-------. 1 qemu qemu 896K Oct 11 17:56 /var/lib/libvirt/images/Snap-1.qcow2
         392K -rw-------. 1 qemu qemu 448K Oct 11 17:56 /var/lib/libvirt/images/Snap-2.qcow2
         456K -rw-------. 1 qemu qemu 512K Oct 11 17:56 /var/lib/libvirt/images/Snap-3.qcow2
        1011M -rw-------. 1 qemu qemu 3.0M Oct 11 18:29 /var/lib/libvirt/images/Active.qcow2
```

(5) 检查'Active'信息，现在它已经不需要backing file了，正如*Figure-6*所示::
```
        # qemu-img info /var/lib/libvirt/images/Active.qcow2
        image: /var/lib/libvirt/images/Active.qcow2
        file format: qcow2
        virtual size: 1.0G (1073741824 bytes)
        disk size: 1.0G
        cluster_size: 65536
```
(6) 清理现场
    ::
```
        # virsh snapshot-delete --domain RootBase Snap-3 --metadata
```
(7) 现在还可以使用下 guestfish  **READ-ONLY**  模式来检查下磁盘内容( *--ro* 选项)
    ::
```
        # guestfish --ro -i -a /var/lib/libvirt/images/Active.qcow2
```
快照删除 (and 'offline commit')

删除（live/offline）状态的*内置快照*很方便 ::
```
# virsh snapshot-delete --domain f17vm --snapshotname snap6
```
[OR]
```
# virsh snapshot-delete f17vm snap6
```
libvirt现在还没有删除外置快照的功能，但是可以使用*qemu-img*命令来完成.

比如我们有这样一条链(VM*offline*状态): base <- sn1 <- sn2 <- sn3

现在删除第二个快照(sn2).有两种方式:

* Method (1): base <- sn1 <- sn3 (by copying sn2 into sn1)
* Method (2): base <- sn1 <- sn3 (by copying sn2 into sn3)

Method (1)

(by copying sn2 into sn1)

注意: 必须保证sn1没有被其他快照作为后端,不然就挂了!!

offline commit

```
# qemu-img commit sn2.qcow2
```

将会*commit*所有在sn2中的改动到sn2的backing file(sn1).
qemu-img commit和virsh blockcommit类似
现在把sn3的后端指向到sn1.

```
# qemu-img rebase -u -b sn1.qcow2 sn3.qcow2
```

注意: -u代表'Unsafe mode' -- 此模式下仅仅修改了指向到的backing file名字，必须谨慎操作.
现在可以直接删除sn2
```
# rm sn2.qcow2
```

Method (2)

(by copying sn2 into sn3)

合并数据，rebase后端:
```
# qemu-img rebase -b sn1.qcow2 sn3.qcow2
```
未使用-u模式的rebase将把数据也一并合并过去，即sn2的数据写入到sn3.
换言之: 这里使用的'Safe mode',也是默认模式 --对sn3而言任何从
qemu-img rebase(没有-u)和和virsh blockpull类似.
backingfile（sn1）到旧的backingfile（sn2）之间发生的差异改动都将被合并到sn3中.

现在可以删除sn2了
```
# rm sn2.qcow2
```
