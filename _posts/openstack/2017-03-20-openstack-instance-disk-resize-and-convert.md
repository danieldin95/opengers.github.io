---
title: "openstack底层技术-实例磁盘扩容及格式转换"
author: opengers
layout: post
permalink: /virtualization/openstack-instance-disk-resize-and-convert/
categories: virtualization
tags:
  - openstack
  - kvm
  - convert
format: quote
---

><small>主要总结下虚拟机实例磁盘扩容以及格式间的转换</small>    

本文所使用宿主机，及kvm版本如下     

``` shell
#cat /etc/centos-release
CentOS Linux release 7.2.1511 (Core) 

#rpm -qa |grep qemu
qemu-img-1.5.3-105.el7_2.4.x86_64
qemu-kvm-common-1.5.3-105.el7_2.4.x86_64
qemu-kvm-1.5.3-105.el7_2.4.x86_64
libvirt-daemon-driver-qemu-1.2.17-13.el7_2.5.x86_64
```

## 几种磁盘格式介绍      

**qcow2**  

qcow2磁盘格式是qemu模拟器支持的一种磁盘镜像。它可以用一个文件的形式来表示一块固定大小的块设备磁盘。与普通的raw格式相比，有以下特性

* 更小的空间占用，即使文件系统不支持空洞(holes)     
* 支持写时拷贝（COW, copy-on-write，镜像文件只反映底层磁盘的变化   
* 支持快照（snapshot），镜像文件能够包含多个快照的历史   
* 可选择基于 zlib 的压缩方式     

创建qcow2磁盘格式      

``` shell
#qemu-img create -f qcow2 disk1.qcow2 40G
Formatting 'disk1.qcow2', fmt=qcow2 size=42949672960 encryption=off cluster_size=65536 lazy_refcounts=off 

#qemu-img info disk1.qcow2 
image: disk1.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 196K
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false

#ls -lh disk1.qcow2 
-rw-r--r-- 1 root root 193K Apr 22 16:13 disk1.qcow2
```

**raw**   

raw格式应该是最常见的一种磁盘格式，平常所使用的文件都是raw格式    

``` shell
#qemu-img info /etc/passwd
image: /etc/passwd
file format: raw
virtual size: 2.0K (2048 bytes)
disk size: 4.0K

#qemu-img create -f raw disk2.raw 40G
Formatting 'disk2.raw', fmt=raw size=42949672960 

#qemu-img info disk2.raw 
image: disk2.raw
file format: raw
virtual size: 40G (42949672960 bytes)
disk size: 0

#ls -lh disk2.raw 
-rw-r--r-- 1 root root 40G Apr 22 16:15 disk2.raw
```

**RBD**    

rbd磁盘是ceph提供的块设备，可以在其上安装系统，或者挂载到实例上作为数据卷     

``` shell
#qemu-img create rbd:vms/disk3.rbd 40G
Formatting 'rbd:vms/disk3.rbd', fmt=raw size=42949672960 cluster_size=0 

#qemu-img info rbd:vms/disk3.rbd
image: rbd:vms/disk3.rbd
file format: raw
virtual size: 40G (42949672960 bytes)
disk size: unavailable
```    

## 关于磁盘设备与文件系统    

扩容前，我们先了解下磁盘设备与文件系统，这是了解磁盘扩容的基础        

考虑下我们安装虚拟机的过程，新建一块制定大小的磁盘设备，然后在其上新建分区，安装系统。刚新建磁盘设备时，磁盘上并无任何数据，之后新建分区会向磁盘上写入inode, 一旦分区建立，inode数量就也确定，文件系统就形成了。关于inode，可以参考这里[理解inode](http://www.ruanyifeng.com/blog/2011/12/inode.html)    

假如有一台40G根磁盘的centos6虚拟机需要扩容到80G。首先关机，然后使用`qemu-img resize disk +40G`扩容系统盘到80G，启动虚拟机，使用`fdisk`命令可以看到磁盘大小为80G，然而使用`df -h`查看却仍为40G，并没有增大。原因就很清楚了，虽然磁盘大小增大了，但是虚拟机并不会自动扩容文件系统的inode以匹配这新增的40G空间，因此真正能供我们写文件的大小仍为40G    

我们需要利用工具使文件系统能够检测磁盘大小变化，并增大inode数量   

## centos6虚拟机扩容           

centos6磁盘格式为ext4，centos7系列使用的xfs，因此扩容方式也不同      

测试用的虚拟机为centos6.3系统，磁盘如下，我们准备把根磁盘扩容到80G。扩容主要是使用growpart等工具，在系统启动时检测磁盘大小变化，自动增大文件系统inode数量    

``` shell    
#virsh domblklist centos6
Target     Source
------------------------------------------------
vda        /data/test/centos6.qcow2

#qemu-img info /data/test/centos6.qcow2
image: /data/test/centos6.qcow2
file format: qcow2
virtual size: 40G (85899345920 bytes)
disk size: 1.2G

```

首先关闭虚拟机，resize磁盘到80G    

``` shell
virsh shutdown centos6

qemu-img resize centos6.qcow2 +40G

rbd resize vms/disk.rbd -s 400G
```

启动虚拟机，会发现磁盘大小已变为80G，下面我们安装工具扩容其文件系统，需要注意的是，因为根磁盘在虚拟机运行期间处于挂载状态，无法扩容，只能在系统启动期间让工具自动检测并扩容     
  
``` shell

#安装parted growpart
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/cloud-utils-growpart-0.27-10.el6.x86_64.rpm
yum install parted

wget https://github.com/flegmatik/linux-rootfs-resize/archive/master.zip 
unzip master
cd linux-rootfs-resize-master
./install
```

之后重启虚拟机，启动过程中inode会自动扩容   

![resize](/images/openstack/install-resize/resize1.png)    

## centos7下的磁盘扩容      

centos7中默认使用xfs格式，下面以xfs磁盘格式为例。   

同样，首先关机，扩容虚拟机磁盘     

``` shell
qemu-img resize centos7.qcow2 +40G    
```

然后启动虚拟机    

``` shell
#安装扩容工具  

#yum install cloud-utils-growpart

#磁盘已扩为80G
#fdisk -l

Disk /dev/vda: 85.9 GB, 85899345920 bytes, 167772160 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000b314e

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1            2048    33556479    16777216   82  Linux swap / Solaris
/dev/vda2   *    33556480    83886079    25164800   83  Linux

[root@localhost ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        24G  1.6G   23G   7% /
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           2.0G  8.3M  2.0G   1% /run
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs           396M     0  396M   0% /run/user/0

#我们把40G加到分区2上   
#growpart /dev/vda 2
CHANGED: partition=2 start=33556480 old: size=50329600 end=83886080 new: size=134210315,end=167766795

#扩容分区2文件系统
[root@localhost ~]# xfs_growfs /
meta-data=/dev/vda2              isize=512    agcount=4, agsize=1572800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=6291200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=3071, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 6291200 to 16776289
```


       