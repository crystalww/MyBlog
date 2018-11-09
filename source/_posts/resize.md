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
[root@centos centos]# df -h
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
`[root@centos ~]# tar cvf /tmp/home.tar /home | ls -lh /tmp`
3. 卸载/home目录
终止使用/home文件系统的进程，再卸载。
```
[root@centos ~]# fuser -km /home

[root@centos ~]# umount /home

[root@centos ~]# df -h
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
再次`df -h`可以看到/root大小已经改变了
8. 重新创建home lv
`[root@centos ~]# lvcreate -L 56.43GiB -n /dev/mapper/centos-home`
9. 创建文件系统
`[root@centos ~]# mkfs.xfs /dev/mapper/centos-home`
10. 挂载home
`mount /dev/mapper/centos-home`
11. home文件恢复
`[root@centos /]# tar xvf /tmp/home.tar -C ./`