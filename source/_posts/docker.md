---
title: 搭建docker仓库
date: 2018-10-31 21:09:34
categories:
- Docker
tags:
- centos
- docker
---
在使用kolla部署openstack时要下载很多镜像，部署时间长短容易受网速影响。提前下载所有镜像或在第一次部署完成下载所有镜像后把镜像上传到本地私有docker仓库，可以加快后面再次部署需要的时间。
<!-- more -->
## CentOS7安装docker
要求：系统为64位、内核版本为 3.10 以上。
- 查看你当前的内核版本
`uname -r`
- 安装 Docker
`yum -y install docker`
- 启动 Docker 后台服务
`service docker start`
- 设置开机自启动
`systemctl enable docker`
- 测试运行 hello-world
由于本地没有hello-world这个镜像，所以会下载一个hello-world的镜像，并在容器内运行。
`docker run hello-world`

## 搭建本地仓库
- 获取官方registry镜像
`docker pull registry`
- 启动容器
默认情况下，仓库会被创建在容器的 /var/lib/registry 目录下。你可以通过 -v 参数来将镜像文件存放在本地的指定路径。例如下面的例子将上传的镜像放到本地的 /opt/data/registry 目录。
```
docker run -d -p 5000:5000 --privileged=true --restart=always --name registry -v /opt/data/registry:/var/lib/registry registry
```
- 在本机查看已有的镜像
```
[root@localhost ~]# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
docker.io/registry      latest              2e2f252f3c88        6 weeks ago         33.3 MB
docker.io/hello-world   latest              4ab4c602aa5e        6 weeks ago         1.84 kB
```
- 标记本地镜像
```
[root@localhost ~]# docker tag docker.io/hello-world 192.168.89.149:5000/docker.io/hello-world
[root@localhost ~]# docker images
REPOSITORY                                  TAG                 IMAGE ID            CREATED             SIZE
docker.io/registry                          latest              2e2f252f3c88        6 weeks ago         33.3 MB
192.168.89.149:5000/docker.io/hello-world   latest              4ab4c602aa5e        6 weeks ago         1.84 kB
docker.io/hello-world                       latest              4ab4c602aa5e        6 weeks ago         1.84 kB
```
- 修改/etc/docker/daemon.json，并重启docker（如果重启失败，将daemon.json重命名为daemon.conf）
```
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "insecure-registries": ["192.168.89.149:5000"]
}

[root@localhost ~]# systemctl daemon-reload 
[root@localhost ~]# systemctl restart docker
```
- 上传标记镜像
```
[root@localhost ~]# docker push 192.168.89.149:5000/docker.io/hello-world
The push refers to a repository [192.168.89.149:5000/docker.io/hello-world]
428c97da766c: Pushed 
latest: digest: sha256:1a6fd470b9ce10849be79e99529a88371dff60c60aab424c077007f6979b4812 size: 524
```
- 用 curl 查看仓库中的镜像。
```
[root@localhost ~]# curl 127.0.0.1:5000/v2/_catalog
{"repositories":["docker.io/hello-world"]}

[root@localhost ~]# curl 127.0.0.1:5000/v2/kolla/centos-source-haproxy/tags/list
{"name":"kolla/centos-source-haproxy","tags":["rocky","ocata"]}
```
- 先删除已有镜像，再尝试从私有仓库中下载这个镜像。
```
[root@localhost ~]# docker rmi 192.168.89.149:5000/docker.io/hello-world

[root@localhost ~]# docker pull 192.168.89.149:5000/docker.io/hello-world
```

## 将下载好的openstack镜像存储到仓库中
```
# 标记镜像
docker images | grep -v TAG | awk '{print $1,$2}' OFS=':' | xargs -I {} docker tag {} '192.168.89.149:5000/'{}
# 上传镜像
docker images | grep -v TAG | grep 192 | awk '{print $1,$2}' OFS=':' | xargs -I {} docker push {}
```