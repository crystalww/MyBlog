---
title: 使用kolla部署openstack（一）
date: 2018-10-29 20:44:24
categories:
- OpenStack
tags:
- openstack
- kolla
- centos
---
## 实验环境
最近在折腾kolla部署容器化的openstack，遇到的坑实在是太多了，在虚拟机中把可能遇到的问题都查了一遍才敢部署到物理机上。但还是没搞明白为什么虚拟机中virt_type只能为qemu才可以创建instance。有兴趣的建议先在VMware Workstation中尝试<!-- more -->，处理器开启`虚拟化Intel VT-x/EPT`或`AMD-V/RVI(V)`，所有操作在root下进行。虚拟机配置如下：
```
系统：CentOS 7
内存：16GB
处理器：4 
硬盘：100GB
网卡1：桥接模式
网卡2：桥接模式
```

修改CentOS默认yum源
>1 备份默认yum源
`mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup`
>2 下载阿里云的yum源配置文件到/etc/yum.repos.d/
`wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo`
>3 本地缓存服务器上的软件包信息
`yum makecache`
>4 更新
`yum -y update`

修改 ~/.pip/pip.conf（没有就创建一个）
```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

修改/etc/hostname和/etc/hosts
>hosts中写用到的主机的ip和hostname，部署用到的每台主机都要改。

关闭selinux
>临时关闭：setenforce 0
永久关闭：改配置文件/etc/selinux/config,将其中SELINUX设置为disabled。

关闭防火墙
```
systemctl status firewalld
systemctl stop firewalld
systemctl disable firewalld
```
---
以上步骤完成后就可以按照[OpenStack Docs的Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)（For development）部分进行操作了。如果是多节点部署，在deploy主机上执行以下命令：
```
# 安装pip并更新到最新
yum install -y epel-release
yum install -y python-pip
pip install -U pip

# 安装依赖
yum install -y python-devel libffi-devel gcc openssl-devel libselinux-python

# 安装ansible
yum install -y ansible

# 更新ansible到最新
pip install -U ansible

# 在ansible配置文件/etc/ansible/ansible.cfg中增加信息（可选）
[defaults]
host_key_checking=False
pipelining=True
forks=100

# 安装Kolla-ansible
git clone https://github.com/openstack/kolla -b stable/rocky
git clone https://github.com/openstack/kolla-ansible -b stable/rocky

# 安装kolla和kolla-ansible的requirements
pip install -r kolla/requirements.txt
pip install -r kolla-ansible/requirements.txt

# 把kolla-ansible的配置文件（globals.yml、passwords.yml）复制到/etc/kolla中
mkdir -p /etc/kolla
cp -r kolla-ansible/etc/kolla/* /etc/kolla

# 复制（all-in-one、multinode）到当前目录下
cp kolla-ansible/ansible/inventory/* .
```
---
## 修改配置信息
单节点部署修改all-in-one文件：
```
[control]
localhost       ansible_connection=local

[network]
localhost       ansible_connection=local

[external-compute]
localhost       ansible_connection=local

[storage]
localhost       ansible_connection=local

[monitoring]
localhost       ansible_connection=local

[deployment]
localhost       ansible_connection=local
```
多节点部署修改mutinode文件：
```
[control]
# These hostname must be resolvable from your deployment host
control2 ansible_user=root ansible_password=123 ansible_become=true

[network]
control2 ansible_user=root ansible_password=123 ansible_become=true

[external-compute]
control2 ansible_user=root ansible_password=123 ansible_become=true

[monitoring]
control2 ansible_user=root ansible_password=123 ansible_become=true

[storage]
control2 ansible_user=root ansible_password=123 ansible_become=true

[deployment]
localhost       ansible_connection=local ansible_become=true
```

检查配置文件信息是否正确
>ansible -i multinode all -m ping

生成随机密码，密码存放于`/etc/kolla/passwords.yml`中。修改此文件中`keystone_admin_password`字段值为`admin`。
>./kolla-ansible/tools/generate_passwords.py

修改/etc/kolla/globals.yml
```
kolla_base_distro: "centos"
kolla_install_type: "source"
openstack_release: "rocky"       # openstack_release版本与你clone的kolla和kolla-ansible的版本对应
network_interface: "eth0"        # 内网
neutron_external_interface: "eth1"  # 外网
kolla_internal_vip_address: "内网ip"   # 确保ip未被占用，这也是用于访问horizon的ip
nova_console: "spice"
enable_horizon: "yes"
nova_compute_virt_type: "kvm"  # 如果是虚拟机，只能用qemu代替kvm，否则实例启动会卡在：Booting from Hard Disk...
docker_registry: "ip:port"     # 如果有私有docker仓库，部署过程会更快

enable_cinder: "yes"           # 块存储，这里还不怎么懂！！！
enable_cinder_backend_lvm: "yes"

# 如果要安装ceilometer、aodh、gnocchi，需要修改下面这部分
enable_aodh: "yes"
enable_ceilometer: "yes"
enable_ceph: "no"
enable_gnocchi: "yes"
ceilometer_database_type: "gnocchi"
ceilometer_event_type: "gnocchi"
gnocchi_backend_storage: "{{ 'ceph' if enable_ceph|bool else 'file' }}"

# 如果要安装mistral，需要修改下面这部分
enable_horizon_mistral: "{{ enable_mistral | bool }}"
enable_mistral: "yes"

```

在下面Bootstrap servers步骤后，修改被部署openstack机器上/etc/docker/daemon.json，并重启docker
```
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "insecure-registries": ["192.168.89.149:5000"]
}
```
---
## 部署
```
# 1. Bootstrap servers with kolla deploy dependencies
./kolla-ansible/tools/kolla-ansible -i multinode bootstrap-servers

# 2. 对主机进行部署前检查（kolla_internal_vip_address如果被占用会失败）
./kolla-ansible/tools/kolla-ansible -i multinode prechecks

# 3. 进行openstack部署
./kolla-ansible/tools/kolla-ansible -i multinode deploy

# 4. 如果部署失败，销毁失败的部署
./kolla-ansible/tools/kolla-ansible -i <<inventory-file>> destroy
```
部署成功后输入虚拟ip地址即可进入dashboard。

## 扩展节点
如果增加了几台服务器用作计算节点或者存储节点，要怎么部署进去呢？
- 进行部署操作的第一、二步，在此过程可能会卡在安装pip这里，进服务器手动安装`yum install -y python-pip`即可。
- 然后进行upgrade而不是重新deploy（验证：如果启动了虚拟机，upgrade没有用）
`./kolla-ansible/tools/kolla-ansible -i multinode upgrade`

---
## kolla部署OpenStack Ocata版本注意事项
- 安装完后需要在/etc/kolla/nova-compute/nova.conf中`[libvirt]`下添加`virt_type = qemu`。
- 确保selinux关闭，否则部署完成实例创建失败
- 修改kolla-ansible/ansible/roles/haproxy/tasks/precheck.yml中的`always_run: True`为`run_once: true`，否则报错：
```
TASK [haproxy : include] *************************************************************************************
fatal: [vmc]: FAILED! => {"reason": "'always_run' is not a valid attribute for a Task\n\nThe error appears to have been in '/root/kolla-ansible/ansible/roles/haproxy/tasks/precheck.yml': line 9, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: Clearing temp kolla_keepalived_running file\n  ^ here\n\nThis error can be suppressed as a warning using the \"invalid_task_attribute_failed\" configuration"}
```
- 确保部署openstack的机器上libvirtd停止运行`service libvirtd status`，`service libvirtd stop`，否则报错：
```
TASK [nova : Checking that libvirt is not running] ***********************************************************
fatal: [vmc]: FAILED! => {"changed": false, "failed_when_result": true, "stat": {"atime": 1541046790.1479998, "attr_flags": "", "attributes": [], "block_size": 4096, "blocks": 0, "charset": "binary", "ctime": 1541046790.1479998, "dev": 19, "device_type": 0, "executable": true, "exists": true, "gid": 0, "gr_name": "root", "inode": 37921, "isblk": false, "ischr": false, "isdir": false, "isfifo": false, "isgid": false, "islnk": false, "isreg": false, "issock": true, "isuid": false, "mimetype": "inode/socket", "mode": "0777", "mtime": 1541046790.1479998, "nlink": 1, "path": "/var/run/libvirt/libvirt-sock", "pw_name": "root", "readable": true, "rgrp": true, "roth": true, "rusr": true, "size": 0, "uid": 0, "version": null, "wgrp": true, "woth": true, "writeable": true, "wusr": true, "xgrp": true, "xoth": true, "xusr": true}}
        to retry, use: --limit @/root/kolla-ansible/ansible/site.retry
```
- 修改kolla-ansible/ansible/roles/mariadb/tasks/lookup_cluster.yml文件，删除所有的`always_run: True`，否则报错：
```
TASK [mariadb : include] *************************************************************************************
fatal: [vmc]: FAILED! => {"reason": "'always_run' is not a valid attribute for a Task\n\nThe error appears to have been in '/root/kolla-ansible/ansible/roles/mariadb/tasks/lookup_cluster.yml': line 2, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n---\n- name: Cleaning up temp file on localhost\n  ^ here\n\nThis error can be suppressed as a warning using the \"invalid_task_attribute_failed\" configuration"}
        to retry, use: --limit @/root/kolla-ansible/ansible/site.retry
```
- 在虚拟机上部署过程中会可能网卡出现问题（ping不通）导致部署失败，重启网卡即可
```
# network
service network restart

# ifdown/ifup
ifdown eth0
ifup eth0

# ifconfig
ifconfig eth0 down
ifconfig eth0 up
```
---
## 调整
>如果需要访问外部网络，
```
供应商类型：Flat
物理网络：physnet1 （/etc/kolla/neutron-server/ml2_conf.ini中flat_networks字段值）
添加内部接口（子网网关）
```
>开启多域支持，修改/etc/kolla/horizon/local_settings，重启horizon容器。
```
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
```
>VMWare虚拟机迁移到openstack：将vmdk转换成qcow2格式的镜像，再上传即可用来创建实例
```
qemu-img convert -f vmdk -O qcow2 Metasploitable.vmdk Metasploitable.img
qemu-img info Metasploitable.img
```
>Horizon登陆超过一小时需要重新登录，改为24小时
修改`/etc/kolla/horizon/local_settings`，添加`SESSION_TIMEOUT = 86400`
修改`/etc/kolla/keystone/keystone.conf`如下，重启horizon、keystone容器
```
[token]
...
expiration = 86400
```

**注意：**
kolla-ansible方式安装的openstack（[P版本及以后](https://github.com/openstack/kolla-ansible/blob/master/releasenotes/notes/keystone-versionless-endpoint-ae9274c81927d949.yaml)）访问API（endpoint）与之前版本不一样，在用openstack4j(V3.1.0)认证时获取不到信息，需要自己修改endpoint。下面操作在deploy上面进行：
```
# Install basic OpenStack CLI clients:
pip install python-openstackclient python-glanceclient python-neutronclient

# generate an openrc file
cd kolla-ansible/tools
./kolla-ansible post-deploy
. /etc/kolla/admin-openrc.sh

# 查看及修改端点信息
[root@deploy tools]# openstack endpoint list
[root@deploy tools]# openstack endpoint --help
[root@deploy tools]# openstack endpoint set --help

# 查看实例
[root@deploy ~]# nova list

# 查看hypervisor类型
[root@deploy ~]# openstack hypervisor list

# 查看服务列表
[root@deploy ~]# openstack service list
[root@deploy ~]# nova service-list
[root@deploy ~]# openstack compute service list 
```
---
## 报错信息整理
**部署过程报错：**
- 安装kolla依赖时报错：cannot uninstall xxx
>在命令后加上`--ignore-installed xxx`再执行一次

- TASK [haproxy : Waiting for virtual IP to appear] 
>编辑/etc/kolla/globals.yml，重新设置keepalived_virtual_router_id的值(Must be free in the network)。

- waiting for nova-compute service up...
>kolla-ansible版本要与openstack_release版本（docker hub中查看tag）一致。
pip安装的kolla-ansible（pip show kolla-ansible）、git clone的master版都对应最新的openstack稳定版（现在是rocky）

- neutron容器启动报错：超时
>kolla部署过程会断开外网网卡导致下载不到docker镜像。提前下载镜像自建docker仓库或者network_interface对应的网卡能上网就行

- iscsid容器启动不了，docker logs iscsid 显示为: Can not bind IPC socket 
```
计算节点或存储节点iscsid容器启动失败，因为物理主机上的iscsid服务占用了IPC socket

解决方案:
yum remove iscsi-initiator-utils
或者
$ sudo systemctl stop iscsid.socket iscsiuio.socket iscsid.service 
$ sudo systemctl disable iscsid.socket iscsiuio.socket iscsid.service  
```

**创建实例报错：**
- no valid hosts...
```
1. 可能是virt_type=kvm导致的，配置文件/etc/kolla/nova-compute/nova.conf中virt_type改为qemu，重启nova_compute容器
[libvirt]
connection_uri = qemu+tcp://192.168.89.149/system
virt_type = qemu

2. 也可能是其他原因
- nova-compute日志提示
ERROR nova.compute.manager [instance: 3af11e19-b4f8-452a-8f3d-3d659be050bd] libvirtError: Did not receive a reply. Possible causes include: the remote application did not send a reply, the message bus security policy blocked the reply, the reply timeout expired, or the network connection was broken.
- libvirtd日志提示
2018-11-02 01:14:26.017+0000: 43848: error : virDBusCall:1570 : error from service: CanSuspend: Did not receive a reply. Possible causes include: the remote application did not send a reply, the message bus security policy blocked the reply, the reply timeout expired, or the network connection was broken.

### 解决办法
检查节点状态，发现selinux开启了，关闭selinux即可
查看selinux状态：sestatus
临时关闭：setenforce 0
永久关闭：改配置文件/etc/selinux/config,将其中SELINUX设置为disabled。
```

- MessagingTimeout: Timed out waiting for a reply to message ID... 
>查看nova_compute的日志，如果需要重启容器：docker restart $(docker ps -aq)

- CentOS 7 packstack - Instance fails, Error: Volume did not finish being created
```
(nova-compute)[root@m_controller /]# cat /var/log/kolla/nova/nova-compute.log

ERROR nova.compute.manager [instance: 3c122d5f-bd92-45c1-ae11-e6ca743a83b1] BuildAbortException: Build of instance 3c122d5f-bd92-45c1-ae11-e6ca743a83b1 aborted: Volume 4a7a123e-8d90-4365-bb11-24688cb3ad3e did not finish being created even after we waited 0 seconds or 1 attempts. And its status is error.
```
  查看nova_compute的日志，应该是卷的原因，可能是[卷的存储不够](https://ask.openstack.org/en/question/111367/centos-7-packstack-instance-fails-error-volume-did-not-finish-being-created/)，创建实例时默认开启卷，关掉就好了
![cinder](cinder.png)

---
## 参考资料
1. [OpenStack Docs: Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html#kolla-globals-yml)
2. [Docker容器化部署运维OpenStack和Ceph](https://cloud.tencent.com/developer/article/1073885)
3. [CentOS7单节点部署OpenStack-Pike(使用kolla-ansible)](https://blog.csdn.net/persistvonyao/article/details/80229602)
4. [Openstack之ubuntu16使用kolla部署实验](http://blog.51cto.com/yuweibing/1976882)
5. [kolla-ansible安装openstack(Ocata)](https://www.cnblogs.com/silvermagic/p/7665975.html)
6. [Timeout of Horizon web interface when idle](https://openstack.nimeyo.com/101821/openstack-timeout-of-horizon-web-interface-when-idle)