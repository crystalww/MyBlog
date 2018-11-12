---
title: openstack虚拟机鼠标指针问题
date: 2018-11-12 18:30:10
categories:
- OpenStack
tags:
- openstack
---
openstack（Ocata）创建实例，如果是windows系统，桌面会出现两个鼠标，操作不便。原因在于鼠标驱动程序为`PS/2兼容鼠标`，而实际使用的是`usb鼠标`。
<!-- more -->
因为我是用kolla部署的openstack，配置文件与手动安装方式相比略有不同。查阅相关资料，主要有以下几种解决方法：
1. 直接修改镜像，附加属性，再用该镜像生成虚拟机。用kolla部署的话，需要在deploy上操作，安装openstack cli client相关依赖（见上一篇）。
`openstack image set  --property hw_pointer_model=usbtablet 927e8608-768f-4415-871b-0fc2cce79a14` 最后一个参数为镜像id
![image1](image1.png)
修改之后如下
![image2](image2.png)
然后创建虚拟机，发现问题并没有解决~~~

2. 直接修改虚拟机配置文件
先创建一个实例（虚拟机），虚拟机配置文件一般存放在libvirt的qemu文件夹下，是创建虚拟机的时候自动生成的，如`/etc/libvirt/qemu/instance-00000005.xml`。
- 进入nova_libvirt容器
`[root@centos ~]# docker exec -itu root nova_libvirt /bin/bash`
- 修改虚拟机配置文件
`(nova-libvirt)[root@centos /]# vi /etc/libvirt/qemu/instance-00000005.xml`
定位到`<input>`标签，在前面添加`<input type='tablet' bus='usb'/>`，保存的时候你会发现报错：`"internal error: No free USB ports"`。
对比手动安装的，应该要添加的语句如下，然后删掉`<input type='mouse' bus='ps2'/>`。
```
<input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
</input>
```
- 保存配置后重启容器
`[root@centos ~]# docker restart nova_libvirt`
软重启实例，然后鼠标正常了。但是这样有个缺点，每次创建都要改配置文件非常麻烦，而且配置文件也不建议修改（动态生成，会被覆盖掉）。

3. 修改nova配置文件（推荐）
- 修改/etc/kolla/nova-compute/nova.conf文件
在`[DEFAULT]`下添加`pointer_model = usbtablet`
[官方文档](https://docs.openstack.org/ocata/config-reference/compute/config-options.html)中有这么一段话
```
usbtablet must be configured with VNC enabled or SPICE enabled and SPICE agent disabled. When used with libvirt the instance mode should be configured as HVM.
```
- 在`[spice]`下添加`agent_enabled=false`。
忽略这一步导致浪费了很多时间啊！！！
![usbtablet](usbtablet.png)
- 重启nova_compute容器
`[root@centos ~]# docker restart nova_compute`

## 参考资料
1. [openstack 填坑笔记4：windows 实例运行出现两个鼠标，重影难于对焦](https://blog.csdn.net/oLinBSoft/article/details/80320322)
2. [use_usb_tablet=true have no effect](https://bugs.launchpad.net/nova/+bug/1356633)