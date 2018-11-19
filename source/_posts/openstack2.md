---
title: openstack镜像操作
date: 2018-11-14 10:56:25
categories:
- OpenStack
tags:
- openstack
---
在部署好openstack后需要镜像来创建实例，CentOS、Ubuntu等系统有[官方维护的镜像](https://docs.openstack.org/image-guide/obtain-images.html)，windows系统的镜像则需要自己制作。下面是我在CentOS7上制作XP系统和win7系统的记录。
<!-- more -->
## 环境准备
关闭selinux
>查看selinux状态：`sestatus`
临时关闭：`setenforce 0`
永久关闭：`vi /etc/selinux/config`，设置`SELINUX=disabled`
显示当前SELinux的应用模式（强制、执行还是停用）：`getenforce`
检测系统是否支持KVM：`egrep -o '(vmx|svm)' /proc/cpuinfo`

关闭防火墙
>systemctl status firewalld
systemctl stop firewalld
systemctl disable firewalld

安装必要的软件
>yum install -y virt-install virt-manager libvirt qemu-kvm tree

修改/etc/libvirt/qemu.conf
>vnc_listen ="0.0.0.0"

开启libvirtd
>systemctl start libvirtd
systemctl enable libvirtd

创建目录，便于合理存储资源
```
[root@dep centos]# mkdir -p /var/lib/libvirt/images/{iso,virtual}

[root@dep centos]# tree /var/lib/libvirt/images/
/var/lib/libvirt/images/
├── iso
└── virtual
```

创建空磁盘
```
[root@dep centos]# cd /var/lib/libvirt/
[root@dep libvirt]# qemu-img create -f qcow2 images/virtual/winxp.qcow2 10G
[root@dep libvirt]# qemu-img create -f qcow2 images/virtual/win7_x64.qcow2 10G
```

把xp系统和Virtio的iso镜像上传到服务器中
>window和centos系统间传文件可以用[`winscp`](https://winscp.net/eng/download.php)工具。

```
[root@dep libvirt]# tree images/
images/
├── iso
│   ├── virtio-win-0.1.126_x86.vfd
│   ├── virtio-win-0.1.141.iso
│   ├── win7_professional_x64.iso
│   └── zh-hans_windows_xp_professional_with_service_pack_3_x86_cd_x14-80404.iso
└── virtual
    ├── win7_x64.qcow2
    └── winxp.qcow2
```
---
## 制作xp系统
官方给的是windows server2012的封装，照着它的顺序做，Boot Option里面没有系统镜像，这里参考了[《4.2 安装第一个KVM虚拟化Guest操作系统》](https://cloud-atlas.readthedocs.io/zh_CN/latest/c04/p02_install_first_kvm_guest_os.html)。XP的安装程序只支持从软盘加载驱动程序，因此需要一个软盘的镜像文件，在KVM启动的时候进行加载。
```
[root@test centos]# virt-install --connect qemu:///system \
--name winxp --ram 2048 --vcpus 2 \
--network network=default,model=virtio \
--disk path=images/virtual/winxp.qcow2,format=qcow2,device=disk,bus=virtio \
--disk path=images/iso/virtio-win-0.1.126_x86.vfd,device=floppy \
--disk path=images/iso/zh-hans_windows_xp_professional_with_service_pack_3_x86_cd_x14-80404.iso,device=cdrom \
--disk path=images/iso/virtio-win-0.1.141.iso,device=cdrom \
--boot cdrom,hd,menu=on \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--os-type windows --os-variant winxp
```
>`--boot`选项可以通过`man virt-install`查询到。它指定机器的启动顺序，`cdrom`是光驱，`hd`为硬盘，`menu`是`bios boot menu`。
>`--graphics vnc`这一行表示可以用vnc远程连接

- 获取vnc端口号
```
[root@ocata mouri]# virsh vncdisplay winxp
:0

[root@dep centos]# virsh domdisplay winxp
vnc://localhost:0
```
- 用vncviewer连接，输入centos的ip+端口号，如我的为：192.168.89.111:0。

如果centos是图形界面的，也可以用virt-manager和virt-viewer来安装系统。下面的命令也可用：
```
# 若本机不支持虚拟化，请将virt-type选项更改为qemu
[root@ocata libvirt]# virt-install --virt-type kvm \
--name winxp --ram 2048 --vcpus 2 \
--network network=default,model=virtio \
--disk path=images/virtual/winxp.qcow2,format=qcow2,device=disk,bus=virtio \
--disk path=images/iso/virtio-win-0.1.126_x86.vfd,device=floppy \
--disk path=images/iso/zh-hans_windows_xp_professional_with_service_pack_3_x86_cd_x14-80404.iso,device=cdrom \
--disk path=images/iso/virtio-win-0.1.141.iso,device=cdrom \
--os-type=windows --os-variant=winxp \
--graphics vnc,listen=0.0.0.0 --noautoconsole
```
安装过程可能会重启，我的关机之后没有自己启动，需要手动开启，再用VNC Viewer连接。
```
[root@ocata libvirt]# virsh list --all
 Id    名称                         状态
----------------------------------------------------
 -     winxp                          关闭

[root@ocata libvirt]# virsh start winxp
域 winxp 已开
```
安装完成进入计算机管理把驱动程序安装好，选择驱动程序镜像所在盘自动安装即可。
![xp](xp.png)
下一步就是安装[cloudbase-init](https://cloudbase.it/cloudbase-init/)，问题是怎么把文件传入到winxp系统中，可以参考[虚拟机镜像管理工具-- Libguestfs](https://www.jianshu.com/p/c154f43b7fac)。
但是centos7中guestmount已经不支持NTFS的镜像了[(mount: unsupported filesystem type” with NTFS in RHEL ≥ 7.2)](https://forums.urbackup.org/t/mount-image-is-not-working-under-rhel-7-3/3127/2)。也可以在xp系统中上网下载，但是IE半天打不开网页，给换了世界之窗的浏览器。
打开IE-->工具-->Internet选项-->高级-->勾选使用TLS1.0。然后输入世界之窗浏览器的网址下载。

>安装过程中选项：
Username：Administrator；
Serial port for logging：COM1；
勾选Run Cloudbase-init service as LocalSystem；
耐心等待安装完成

并没有Run Sysprep与Shutdown可以勾选，用安装镜像中的support-->tools下的sysprep封装，但是没有可用的密钥，到这一步封装进行不下去了。

## win7系统制作
```
[root@test centos]# virt-install --connect qemu:///system \
--name win7_x64 --ram 2048 --vcpus 2 \
--network network=default,model=virtio \
--disk path=images/virtual/win7_x64.qcow2,format=qcow2,device=disk,bus=virtio \
--disk path=images/iso/win7_professional_x64.iso,device=cdrom  \
--disk path=images/iso/virtio-win.iso,device=cdrom \
--boot cdrom,hd,menu=on \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--os-type windows --os-variant win7
```
也可用下面的命令
```
# 若本机不支持虚拟化，请将<virt-type>选项更改为<qemu>
[root@test centos]# virt-install --virt-type kvm \
--name win7_x64 --ram 2048 --vcpus 2 \
--network network=default,model=virtio \
--disk path=images/virtual/win7_x64.qcow2,format=qcow2,device=disk,bus=virtio \
--disk path=images/iso/win7_professional_x64.iso,device=cdrom  \
--disk path=images/iso/virtio-win.iso,device=cdrom \
--boot cdrom,hd,menu=on \
--graphics vnc,listen=0.0.0.0 --noautoconsole
```
>1. 现在安装
>2. 自定义
>3. 加载驱动程序，选择E盘（virtio所在磁盘）下的visitor对应的系统的amd64或x86文件夹，加载驱动完成即可检测到磁盘
>4. 安装完成后，会自动重启，设置账户及密码，进入桌面系统。此时，虚拟机还需要安装网卡等驱动，在设备管理器中更新驱动程序，选择VirtIO驱动位置更新即可。

win7鼠标重影，先关机，找到配置文件
```
[root@dep libvirt]# find / -name "win7.xml"
/etc/libvirt/qemu/win7.xml
```
添加usbtablet
```
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'/>
```
然后[导入配置](http://www.itshuji.com/technical-article/1774.html)`[root@dep libvirt]# virsh define /etc/libvirt/qemu/win7.xml`，重启使配置生效。

安装cloudinit，完成后勾选Run Sysprep与Shutdown，点击Finish；
```
# 使虚拟机脱离libvirt管理
[root@dep libvirt]# virsh undefine win7_x64
# libguestfs-tools的安装
[root@dep libvirt]# yum -y install libguestfs-tools
# 消除映像空洞
[root@dep libvirt]# virt-sparsify -x images/virtual/winxp.qcow2 --convert qcow2 images/virtual/winxp.qcow2.tmp

[root@dep libvirt]# virt-sparsify -x images/virtual/win7_x64.qcow2 --convert qcow2 images/virtual/win7_x64.qcow2.tmp
# 压缩映像
[root@dep libvirt]# qemu-img convert -c -p -O qcow2 images/virtual/winxp.qcow2.tmp images/virtual/winxp.img

[root@dep libvirt]# qemu-img convert -c -p -O qcow2 images/virtual/win7_x64.qcow2.tmp images/virtual/win7_x64.img
```
```
# 将映像重新加入libvirt管理
virt-install --name win7_x64 \
  --virt-type kvm \
  --ram 2048 \
  --vcpus=2 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --import \
  --disk images/virtual/win7_x64.qcow2,format=qcow2
  ```

---
## 常用命令
```shell
# 查看网络
virsh net-list

# 列出机器所有已安装的虚拟机
virsh list --all
    
# 启动虚拟机
virsh start ubuntu

# 关闭虚拟机
virsh shutdown ubuntu

# 重启虚拟机
virsh reboot ubuntu

# 删除虚拟机和创建的镜像文件
virsh destroy winxp
virsh undefine winxp
rm winxp.qcow2
```
---
## 踩坑记录
>1 [（virt-install）ERROR Network not found: no network with matching name 'default'](https://blog.csdn.net/qq_21398167/article/details/47777113)
- 查找与`network`和`libvirt`有关的`default.xml`
`[root@ocata libvirt]# find / -name "default.xml"`
- 从XML文件中定义一个网络
`[root@ocata libvirt]# virsh net-define /usr/share/libvirt/networks/default.xml`
- 启动`default`网络
`[root@ocata libvirt]# virsh net-start default`
- 查看网络是否启动成功
`[root@ocata libvirt]# virsh net-list`

>2 centos中`virsh vncdisplay winxp`返回了vnc端口号，但是物理机中VNC Viewer连接不了centos中的winxp
- [关闭centos中的selinux和防火墙](http://blog.51cto.com/13520705/2055045)，去掉`/etc/libvirt/qemu.conf`中`#vnc_listen = "0.0.0.0"`的注释。

>3 制作win7镜像时通过virt-viewer看到并没有进入安装程序，提示"No bootable device"
- 原因是没有指定光盘启动，可以通过virt-manager工具配置；也可以加上--boot参数，同时如果有多个iso镜像，系统安装包镜像参数要用`--disk path=/path/to/iso,device=cdrom`的形式。

---
## 镜像操作
想知道通过horizon上传的镜像存哪去了，可以查看glance的配置文件
```
[root@centos ~]# cat /etc/kolla/glance-api/glance-api.conf | grep image
filesystem_store_datadir = /var/lib/glance/images/
```
通过`[root@centos ~]# docker inspect glance_api`查看"Mounts"字段
```
"Mounts": [
            {
                "Source": "/etc/localtime",
                "Destination": "/etc/localtime",
                "Mode": "ro",
                "RW": false,
                "Propagation": "rprivate"
            },
            {
                "Name": "glance",
                "Source": "/var/lib/docker/volumes/glance/_data",
                "Destination": "/var/lib/glance",
                "Driver": "local",
                "Mode": "rw",
                "RW": true,
                "Propagation": "rprivate"
            },
            ...
```
可以看到容器中存放image的目录对应的本地目录，查看本地目录下的文件
```
[root@centos ~]# ls -lh /var/lib/docker/volumes/glance/_data/images/
total 29G
-rw-r-----. 1 42415 42415 5.1G Nov 12 18:40 058f6740-3cde-4d4e-826c-416cab9bdf06
-rw-r-----. 1 42415 42415 1.7G Nov 12 20:30 2c5f7617-506b-4b4e-bafe-ec180b77f9e3
-rw-r-----. 1 42415 42415 8.3G Nov 12 20:08 36ebf445-d410-4c30-8650-f46a814f10b3
-rw-r-----. 1 42415 42415 4.4G Nov 11 20:31 60b6654a-b342-4f5e-80df-0b2cd1a7b51d
-rw-r-----. 1 42415 42415  13M Nov  7 16:38 63f0a9f8-7f4c-4243-954e-00446de8a70c
-rw-r-----. 1 42415 42415 2.0G Nov 14 10:06 8e7ebf4e-c2e8-47dc-9d24-d5376946ee43
-rw-r-----. 1 42415 42415 5.3G Nov 11 20:50 98c6b078-6aa3-4759-8013-ad060a0cffe3
-rw-r-----. 1 42415 42415 1.8G Nov  7 16:43 fbef4306-061b-486b-ab44-c567fc2aac10
```
由于我是用部署机（deploy）给另一台机器部署openstatck，所以在deploy上操作：
```
[root@dep ~]# . /etc/kolla/admin-openrc.sh 

# 查看镜像列表
[root@dep ~]# glance image-list
+--------------------------------------+----------------+
| ID                                   | Name           |
+--------------------------------------+----------------+
| 8e7ebf4e-c2e8-47dc-9d24-d5376946ee43 | centos6.5      |
| 60b6654a-b342-4f5e-80df-0b2cd1a7b51d | centos7        |
| 63f0a9f8-7f4c-4243-954e-00446de8a70c | cirros         |
| 36ebf445-d410-4c30-8650-f46a814f10b3 | kali           |
| fbef4306-061b-486b-ab44-c567fc2aac10 | metasploitable |
| 058f6740-3cde-4d4e-826c-416cab9bdf06 | ubuntu16.04    |
| 98c6b078-6aa3-4759-8013-ad060a0cffe3 | win7           |
| 2c5f7617-506b-4b4e-bafe-ec180b77f9e3 | winxp          |
+--------------------------------------+----------------+

# 查看cirros镜像详细信息（[root@dep ~]# openstack image show cirros）
[root@dep ~]# glance image-show 63f0a9f8-7f4c-4243-954e-00446de8a70c
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2018-11-07T08:38:10Z                 |
| disk_format      | qcow2                                |
| id               | 63f0a9f8-7f4c-4243-954e-00446de8a70c |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros                               |
| owner            | 74615a486948450092e5dd0ba9e077e9     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2018-11-07T08:38:10Z                 |
| virtual_size     | Not available                        |
| visibility       | public                               |
+------------------+--------------------------------------+

# 下载cirros镜像到/home目录下
[root@dep ~]# glance image-download --file /home/test.img 63f0a9f8-7f4c-4243-954e-00446de8a70c

# 查看下载镜像的大小
[root@dep ~]# du -sh /home/test.img 
13M     /home/test.img

# 上传镜像
[root@dep ~]# glance image-create --name "test" --disk-format qcow2 --container-format bare --progress < /home/test.img 
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2018-11-14T03:56:24Z                 |
| disk_format      | qcow2                                |
| id               | b5ca651f-b45e-4269-b7c9-ea671744fb85 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | test                                 |
| owner            | 74615a486948450092e5dd0ba9e077e9     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2018-11-14T03:56:25Z                 |
| virtual_size     | Not available                        |
| visibility       | shared                               |
+------------------+--------------------------------------+

# 删除镜像
[root@dep ~]# glance image-delete b5ca651f-b45e-4269-b7c9-ea671744fb85
```

## 参考资料
[1]. [制作OpenStack云主机映像的指南](https://www.xiaocoder.com/2017/06/29/openstack-virtual-image/)
[2]. [KVM 安装windows XP 系统](https://www.cnblogs.com/songfucai/articles/6710150.html)
[3]. [Openstack镜像制作之windows7篇](http://zjzone.cc/index.php/2017/03/10/openstack-jing-xiang-zhi-zuo-zhi-window7/)
[4]. [为openstack制作windows镜像](https://blog.csdn.net/happyteafriends/article/details/48159427)