---
title: centos7下home目录属性变为问号
date: 2019-10-17 13:31:19
categories:
- CentOS
tags:
- centos
---
centos7下home目录执行权限和所属用户等都变为'??'，输命令时报错信息如下：
```shell
ls: cannot access  Input/output error
ls: cannot open directory .: Input/output error
```
硬盘故障，只读或只写，你可以dmesg|grep sd或dmesg|grep error查看下，应该是有详细报错信息的。
<!-- more -->

使用命令`fsck -t xfs -r /home`修复时，提示查看xfs_repair。
解决方法：
1. 执行`df -Th`
![df](df.png)
2. 卸载/home目录
```shell
[root@centos ~]# fuser -km /home

[root@centos ~]# umount /home
```
3. 修复`xfs_repair /dev/mapper/centos-home`
4. 挂载`mount /dev/mapper/centos-home`

## 参考
1. [cannot access Input/output error](https://www.cnblogs.com/Alanf/p/7509268.html)
2. [磁盘坏道，文件属性为问号](http://support-it.huawei.com/docs/zh-cn/fusioninsight-all/maintenance-guide/zh-cn_topic_0045465564.html)