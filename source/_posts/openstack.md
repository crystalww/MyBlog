---
title: 使用kolla部署openstack（一）
date: 2018-10-29 20:44:24
categories:
- OpenStack
tags:
- openstack
- kolla
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

### 修改CentOS默认yum源
- 备份默认yum源
`mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup`
- 下载阿里云的yum源配置文件到/etc/yum.repos.d/
`wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo`
- 本地缓存服务器上的软件包信息
`yum makecache`
- 更新
`yum -y update`

### 修改 ~/.pip/pip.conf（没有就创建一个）
```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

## 准备工作
基本是按照[OpenStack Docs的Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)进行的（For development）。首先修改/etc/hostname和/etc/hosts（部署用到的每台主机都要改）。

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
```

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
检查配置文件信息是否正确
```
ansible -i multinode all -m ping
```
生成随机密码，密码存放于`/etc/kolla/passwords.yml`中。修改此文件中`keystone_admin_password`字段密码为`admin`。
```
./kolla-ansible/tools/generate_passwords.py
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
# 与内网网卡在同一子网中，且ip未被占用
kolla_internal_vip_address: "10.1.0.250"
nova_console: "spice"
enable_horizon: "yes"
# 如果是虚拟机，只能用qemu代替kvm，否则实例启动会卡在：Booting from Hard Disk...
nova_compute_virt_type: "kvm"
```

## 部署
```
# Bootstrap servers with kolla deploy dependencies
./kolla-ansible/tools/kolla-ansible -i multinode bootstrap-servers

# 对主机进行部署前检查（kolla_internal_vip_address如果被占用会失败）
./kolla-ansible/tools/kolla-ansible -i multinode prechecks

# 进行openstack部署
./kolla-ansible/tools/kolla-ansible -i multinode deploy

# 如果部署失败，销毁失败的部署
./kolla-ansible/tools/kolla-ansible -i <<inventory-file>> destroy
```
部署成功后输入虚拟ip地址即可进入dashboard。

**注意：**
kolla-ansible方式安装的openstack访问API（endpoint）与手动安装的不一样，在用openstack4j认证时获取不到信息，需要自己修改endpoint。下面操作在deploy上面进行：
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
```

## 使用两台虚拟机
一台虚拟机上写配置文件（deploy），给另一台虚拟机部署openstack环境。修改mutinode文件：
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

如果有私有docker仓库，部署过程会更快，需要修改/etc/kolla/global.yml
```
docker_registry: "ip:port"
```

## 报错信息整理
部署过程报错：
- cannot uninstall xxx
>查看安装什么东西导致出错，然后pip install XXXX --ignore-installed xxx

- waiting for nova-compute service up...
>kolla-ansible版本要与openstack_release版本（docker hub中查看tag）一致。pip安装的kolla-ansible（pip show kolla-ansible）对应最新的openstack稳定版（现在是rocky）

- neutron容器启动报错：超时
>kolla部署过程会断开外网网卡导致下载不到docker镜像。提前下载镜像自建docker仓库或者network_interface对应的网卡能上网就行

创建实例报错：
- no valid hosts...
>可能是virt_type=kvm导致的，配置文件/etc/kolla/nova-compute/nova.conf中virt_type改为qemu，重启nova_compute容器

- MessagingTimeout: Timed out waiting for a reply to message ID...
>docker restart $(docker ps -aq)

如果需要访问外部网络，
```
供应商类型：Flat
物理网络：physnet1 （/etc/kolla/neutron-server/ml2_conf.ini中flat_networks字段值）
添加内部接口（子网网关）
```

## 参考资料
1. [OpenStack Docs: Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html#kolla-globals-yml)
2. [Docker容器化部署运维OpenStack和Ceph](https://cloud.tencent.com/developer/article/1073885)
3. [CentOS7单节点部署OpenStack-Pike(使用kolla-ansible)](https://blog.csdn.net/persistvonyao/article/details/80229602)
4. [Openstack之ubuntu16使用kolla部署实验](http://blog.51cto.com/yuweibing/1976882)