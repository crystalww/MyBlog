---
title: openstack使用过程遇到的问题
date: 2018-11-27 09:39:48
categories:
- OpenStack
tags:
- openstack
---
报错信息记录。
<!-- more -->
## docker容器相关
### mariadb容器一直在重启中
进入nova-compute容器查看mariadb的日志，报错信息类似于[[ERROR] WSREP: failed to open gcomm backend connection: 131: invalid UUID: 00000000 (FATAL) at gcomm/src/pc.cpp:PC():271](http://itheadaches.com/error-wsrep-failed-open-gcomm-backend-connection-131-invalid-uuid-00000000-fatal-gcommsrcpc-cpppc271/)
解决方法如下：
```
[root@ocata ~]# find / -name "gvwstate.dat"
find: ‘/run/user/1000/gvfs’: Permission denied
/var/lib/docker/volumes/mariadb/_data/gvwstate.dat

[root@ocata ~]# mv /var/lib/docker/volumes/mariadb/_data/gvwstate.dat /var/lib/docker/volumes/mariadb/_data/gvwstate.dat.bak
[root@ocata ~]# docker restart mariadb
```


## 重新给镜像打tag
```
docker images | grep -v TAG | awk '{print $1,$2}' OFS=':' | xargs -I {} ./rename.sh {}
```
shell脚本内容如下：
```
#!/bin/bash

name1=$1
name2=${name1#*/}

echo 'tag from [' $name1 '] to [' $name2 ']'
exec docker tag $name1 $name2


#  这段代码不起作用
if [[ ${name1:0:3} == "192" ]]
then
  echo '** delete image' $name1
  exec docker rmi $name1
fi
```