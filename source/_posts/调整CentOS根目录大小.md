---
title: 调整CentOS根目录大小
date: 2018-11-09 11:24:19
categories:
- Linux
tags:
- centos
---
安装CentOS时使用了默认分区，结果发现root目录空间太小，创建虚拟机实例申请不到磁盘，在此记录如何调整CentOS root目录的大小。
<!-- more -->

1. 查看分区
```
[root@centos centos]# df -Th
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G   23G   28G  46% /
devtmpfs                  16G     0   16G   0% /dev
tmpfs                     16G     0   16G   0% /dev/shm
tmpfs                     16G   21M   16G   1% /run
tmpfs                     16G     0   16G   0% /sys/fs/cgroup
/dev/sda1               1014M  222M  793M  22% /boot
/dev/mapper/centos-home  857G   37M  856G   1% /home
tmpfs                    3.2G   12K  3.2G   1% /run/user/42
```
2. 备份home分区文件
`[root@centos ~]# tar cvf /tmp/home.tar /home`
**注意检查是否备份成功**
3. 卸载/home目录
终止使用/home文件系统的进程，再卸载。
```
[root@centos ~]# fuser -km /home

[root@centos ~]# umount /home

[root@centos ~]# df -Th
```
4. 删除/home所在的lv
`[root@centos ~]# lvremove /dev/mapper/centos-home`
5. 查看空闲磁盘大小
```
[root@centos ~]# vgdisplay 
  --- Volume group ---
  VG Name               centos
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  5
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <930.00 GiB
  PE Size               4.00 MiB
  Total PE              238079
  Alloc PE / Size       18832 / 73.56 GiB
  Free  PE / Size       219247 / 856.43 GiB
  VG UUID               zNMW93-KFCC-qpQB-Txho-d86c-6Rh7-wSU7dK
```
6. 扩展/root所在的lv
`[root@centos ~]# lvextend -L +800G /dev/mapper/centos-root`
7. 扩展/root文件系统
`[root@centos ~]# xfs_growfs /dev/mapper/centos-root`
再次`df -Th`可以看到/root大小已经改变了
8. 重新创建home lv
`[root@centos ~]# lvcreate -L 56.43GiB -n /dev/mapper/centos-home`
9. 格式化逻辑卷
`[root@centos ~]# mkfs.xfs /dev/mapper/centos-home`
10. 挂载home
`mount /dev/mapper/centos-home`
11. home文件恢复
`[root@centos ~]# tar xvf /tmp/home.tar -C /`


## lvm扩容
如果当前磁盘容量不足，则需要添加新的硬盘。假设新增磁盘为/dev/sdb。
```
[root@localhost ~]# fdisk /dev/sdb
// 打印分区表
Command (m for help): p 

// 大容量磁盘分区表格式改为gpt
Command (m for help): g
Building a new GPT disklabel

// 修改分区格式（改之前先查看，找到Linux LVM的代码）
Command (m for help): t
Selected partition 1
Partition type (type L to list all types): L
// 改为LVM
Partition type (type L to list all types): 31
Changed type of partition 'Linux filesystem' to 'Linux LVM'

// 操作写入分区表
Command (m for help): w
The partition table has been altered!

[root@localhost ~]# partprobe            // 重读分区表
```

创建pv
`pvscan`查看
```
[root@localhost ~]# pvscan 
  PV /dev/sda3   VG centos          lvm2 [<1.82 TiB / 4.00 MiB free]
  Total: 1 [<1.82 TiB] / in use: 1 [<1.82 TiB] / in no VG: 0 [0   ]

[root@localhost ~]# pvcreate /dev/sdb1
WARNING: ext4 signature detected on /dev/sdb1 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/sdb1.
  Physical volume "/dev/sdb1" successfully created.

[root@localhost ~]# pvscan 
  PV /dev/sda3   VG centos          lvm2 [<1.82 TiB / 4.00 MiB free]
  PV /dev/sdb1                      lvm2 [<1.82 TiB]
  Total: 2 [<3.64 TiB] / in use: 1 [<1.82 TiB] / in no VG: 1 [<1.82 TiB]
```

加入vg
```
[root@localhost ~]# vgextend centos /dev/sdb1

[root@localhost ~]# pvscan
  PV /dev/sda3   VG centos          lvm2 [<1.82 TiB / 4.00 MiB free]
  PV /dev/sdb1   VG centos          lvm2 [<1.82 TiB / <1.82 TiB free]
  Total: 2 [<3.64 TiB] / in use: 2 [<3.64 TiB] / in no VG: 0 [0   ]

[root@localhost ~]# vgdisplay 
  --- Volume group ---
  VG Name               centos
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  11
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                3
  Open LV               3
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               <3.64 TiB
  PE Size               4.00 MiB
  Total PE              953556
  Alloc PE / Size       476624 / <1.82 TiB
  Free  PE / Size       476932 / <1.82 TiB
  VG UUID               369M3u-W4jX-vqgv-U3IX-PF9a-yZ87-ThvIPF
```

扩展lv
```
# 显示系统上面lv的状态
[root@localhost ~]# lvdisplay 

# 在lv里面增加容量
[root@localhost ~]# lvextend -L +200G /dev/mapper/centos-home

# 重设lv大小
[root@localhost ~]# xfs_growfs /dev/mapper/centos-home
```
