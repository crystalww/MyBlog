---
title: 使用kolla部署openstack
date: 2018-10-29 20:44:24
categories:
- 
tags:
---
## 实验环境
在VMware workstation中安装centos7系统，处理器开启`虚拟化Intel VT-x/EPT或AMD-V/RVI(V)`。安装好centos系统首先更换国内源，同时pip源更更换为国内源，后面所有操作在root下进行。虚拟机配置如下：
```
内存：16GB
处理器：4
硬盘：100GB
网卡1：桥接模式
网卡2：桥接模式
```
<!-- more -->

### 修改CentOS默认yum源为mirrors.aliyun.com
- 备份系统自带yum源配置文件/etc/yum.repos.d/CentOS-Base.repo<br>
`mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup`
- 下载ailiyun的yum源配置文件到/etc/yum.repos.d/<br>
`wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo`
- 运行yum makecache生成缓存<br>
`yum makecache`
- 更新<br>
`yum -y update`

### 修改 ~/.pip/pip.conf（没有就创建一个）
```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

## 准备工作
基本是按照![OpenStack Docs的Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)进行的。
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

# 在ansible配置文件中增加信息/etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
```
**注意：**
更新ansible可能会遇到`can't uninstall xxx`，只需执行`pip install kolla-ansible --ignore-installed xxx`

```
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

## 修改配置信息
修改all-in-one如下：
```

```
检查配置文件信息是否正确
```
ansible -i multinode all -m ping
```
生成随机密码（存于/etc/kolla/passwords.yml），并修改keystone_admin_password字段密码为 admin
```
cd kolla-ansible/tools
./generate_passwords.py
```
修改/etc/kolla/globals.yml
```
kolla_base_distro: "centos"
kolla_install_type: "source"
openstack_release: "rocky"
openstack_release: "master"
# 内网
network_interface: "eth0"
# 外网
neutron_external_interface: "eth1"
kolla_internal_vip_address: "10.1.0.250"

nova_console: "spice"
enable_horizon: "yes"
nova_compute_virt_type: "qemu"
```
**注意：**

## 部署
```
# Bootstrap servers with kolla deploy dependencies
cd kolla-ansible/tools
./kolla-ansible -i ../ansible/inventory/multinode bootstrap-servers

# 对主机进行部署前检查
./kolla-ansible -i ../ansible/inventory/multinode prechecks

# 进行openstack部署
./kolla-ansible -i ../ansible/inventory/multinode deploy

# 如果部署失败，销毁失败的部署
kolla-ansible destroy -i <<inventory-file>>
```

## 使用两台虚拟机
一台虚拟机上写配置文件，给另一台虚拟机部署openstack环境。修改mutinode文件：
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

如果有私有docker仓库，修改/etc/kolla/global.yml
```
docker_registry: "192.168.89.102:5000"
```

## 参考资料
![OpenStack Docs: Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html#kolla-globals-yml)
![Docker 容器化部署运维 OpenStack 和 Ceph](https://cloud.tencent.com/developer/article/1073885)


![](https://blog.csdn.net/persistvonyao/article/details/80229602)
![](http://blog.51cto.com/yuweibing/1976882)