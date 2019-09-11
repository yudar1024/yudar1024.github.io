+++
author = "陈 sir"
categories = ["kubernetes"]
tags = ["kubernetes"]
date = "2019-09-11"
description = "using heketi to maintain glusterfs "
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "使用 heketi 维护 glusterfs"
type = "post"

+++

glusterfs 最日常的运维工作就是添加磁盘，添加节点。

## 添加磁盘

1. 识别磁盘

linux 生产环境，不一定能够随时停机，此时就要求新添加的硬盘能够被热识别。如果是x86体系的物理硬件，必须断电停机，然后加硬盘。所以不存在问题，但是现在大多数环境是虚拟环境，所以添加硬盘非常方便。不需要断电，也就需要系统能够热识别。

此时linux服务器已经有sda 与sdb 两块磁盘，添加sdc磁盘（未识别，sdc是后面会识别出来的号）

``` shell
# fdisk -l # 查看当前设备状态。此时看不到新添加的设备sdc

# #进入/sys/class/scsi_host目录，在/sys/class/scsi_host下找到符合指向本机iscsi设备主机符号链接表
# ls -al  /sys/class/scsi_host
total 0
drwxr-xr-x.  2 root root 0 Nov 12 15:31 .
drwxr-xr-x. 45 root root 0 Nov 12 15:31 ..
lrwxrwxrwx.  1 root root 0 Nov 12 15:31 host0 -> ../../devices/pci0000:00/0000:00:07.1/host0/scsi_host/host0
lrwxrwxrwx.  1 root root 0 Nov 12 15:31 host1 -> ../../devices/pci0000:00/0000:00:07.1/host1/scsi_host/host1
lrwxrwxrwx.  1 root root 0 Nov 12 15:31 host2 -> ../../devices/pci0000:00/0000:00:10.0/host2/scsi_host/host2

# #本目录下有三个host，分别是host0,host1,host2，在这三个host中，需要确定本机是host0，host1还是host2，在确定host号后，通过重置相应的host存储缓存值就可以发现新添加的硬件了
 
# #4.确定需要重置的host号
 
## 利用grep命令,通过过滤smpspi模块的输出值来确定哪个host链接需要重置,分别查看host0,host1,host2，有mptspi模块值输出的，就是本机需要进行重置的host
 
# grep mpt /sys/class/scsi_host/host0/proc_name
 
# grep mpt /sys/class/scsi_host/host1/proc_name
 
## host2的输出值是mptspi,最终确定host2是需要重置存储缓冲值的host
 
# grep mpt /sys/class/scsi_host/host2/proc_name
 
mptspi
 
## 5.确定host后，重置host2的存储缓存值
 
# # 注:echo "- - -" > /sys/class/scsi_host/host2/scan  “ - - - ”定义了存储在host2中扫描内的三个值，本别是，通道号、SCSI目标ID、LUN值,该命令用通配符替换值，以便它可以检测附加到Linux主机上的新变化。
 
# echo "- - -" > /sys/class/scsi_host/host2/scan
 
## 6.再次查看fdisk，发现系统已经发现新添加的磁盘了，实验完成。
```

2. 使用heketi 添加新设备
此时假设heketi 与glusterfs 已经安装调试完毕。假设我们添加硬盘的节点是node3

``` shell
## 1 查找node3 对应的 id
[root@master1 ~]# heketi-cli node list
Id:0f103c3d9fb68e7eb5606cb1600c9627     Cluster:9c483024cc203cde18f4c822fd1c562c
Id:66b351835fb0759904f10a5e5986aef0     Cluster:9c483024cc203cde18f4c822fd1c562c
Id:9603363276d33ba61b76da8ff297588c     Cluster:9c483024cc203cde18f4c822fd1c562c

## 2 通过 node info 命令查看哪一个是node 3的id
[root@master1 ~]# heketi-cli node info 9603363276d33ba61b76da8ff297588c
Node Id: 9603363276d33ba61b76da8ff297588c
State: online
Cluster Id: 9c483024cc203cde18f4c822fd1c562c
Zone: 1
Management Hostname: node3
Storage Hostname: 192.168.10.53
Devices:
Id:14c09884ab66fbf016c303fde6ff3fb9   Name:/dev/sdb            State:online    Size (GiB):4       Used (GiB):1       Free (GiB):3       Bricks:1

## 3 添加设备
[root@master1 ~]# heketi-cli device add --name=/dev/sdc --node=9603363276d33ba61b76da8ff297588c
Device added successfully
[root@master1 ~]# heketi-cli node info 9603363276d33ba61b76da8ff297588c
Node Id: 9603363276d33ba61b76da8ff297588c
State: online
Cluster Id: 9c483024cc203cde18f4c822fd1c562c
Zone: 1
Management Hostname: node3
Storage Hostname: 192.168.10.53
Devices:
Id:14c09884ab66fbf016c303fde6ff3fb9   Name:/dev/sdb            State:online    Size (GiB):4       Used (GiB):1       Free (GiB):3       Bricks:1
Id:2fba7a17a2e9ead4576d17fbe89e4b2c   Name:/dev/sdc            State:online    Size (GiB):4       Used (GiB):0       Free (GiB):4       Bricks:0
```

