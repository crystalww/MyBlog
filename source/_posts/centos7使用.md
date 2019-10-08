---
title: centos7使用
date: 2018-12-17 14:56:52
categories:
- CentOS
tags:
- centos
---
记录centos7使用过程遇到的问题。
<!-- more -->
**1. 配置静态ip后ping域名要等很久才会有回应，ping ip速度很快。**
检查/etc/sysconfig/network-script/ifcfg-eno1中DNS设置。当设置多个DNS ip地址时，若第一个IP对应的DNS服务器无响应（或者根本就不是DNS服务器），Windows会自动跳过，然后以后都会记住这个顺序，从第二个DNS 进行解析。而CentOS则总是从头到尾进行域名解析，且在第一个DNS IP上浪费不少时间，请求不顺后才会跳到第二个去，之后顺利地解析到域名的IP地址。**在CentOS中，第一个DNS IP的设置是相当重要的！请确保第一个DNS是最为可用、最快的IP，那么打开网站也就不会在域名解析上花费大量无谓的时间了。**

---

**2. centos7 安装Teamviewer**
```
$ wget https://download.teamviewer.com/download/linux/teamviewer-host.x86_64.rpm
$ sudo yum install epel-release
$ sudo yum install ./teamviewer-host*.rpm
```
**Note**:During the transition from 7.3 to 7.4 the cr  repository was also required. In case some dependencies can not be satisfied, you can try with `yum install ./teamviewer-host*.rpm --enablerepo="cr"`.

远程连接后分辨率只有1024*768
### 屏幕支持的分辨率情况
```
$ xrandr
Screen 0: minimum 320 x 200, current 3286 x 1080, maximum 32767 x 32767
eDP1 connected primary 1366x768+0+312 (normal left inverted right x axis y axis) 309mm x 173mm
1366x768      60.1*+
1360x768      59.8    60.0
1024x768      60.0
800x600        60.3    56.2
640x480        59.9
DP1 disconnected (normal left inverted right x axis y axis)
HDMI1 disconnected (normal left inverted right x axis y axis)
DP2 connected 1024x768 (normal left inverted right x axis y axis) 0mm x 0mm
1024x768      60.0*
800x600        60.3    56.2
848x480        60.0
640x480        59.9
HDMI2 disconnected (normal left inverted right x axis y axis)
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
```
- 标示 * 即为屏幕当前分辨率
- 这里显示好几个接口：eDP1, DP1, HDMI1, DP2, HDMI2, VIRTUAL1，但是只有 eDP1 和 DP2 有连接，并且 DP2 对应我们外接屏（这个值后面会用到！）

### 利用cvt新建一个modeline
$ cvt 1920 1080

然后屏幕上会返回两行内容，复制第二行中 'Modeline' 后面的所有内容，并接到下面 xrandr --newmode 后面：
```
$ xrandr --newmode "1920x1080_60.00" 173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync
$ xrandr --addmode DP2 "1920x1080_60.00"
```
其中 ”DP2“ 即上面展示的外接端口，不用的接口这个名字可能不同，比如有的会是 VGA1，以上面 xrandr 的显示结果为准。

之后，再进入 Setting->Displays, 发现那个 “Unknown Display” 的分辨率中，有了 "1920x1080" 这个选项，选中它，并 Apply 即可。或者调用以下命令
```
xrandr --output DP2 --mode "1920x1080_60.00"
```

### 添加开机启动（未实现）
上面的修改重启后就没了，所以写一个脚本如下：/opt/script/myscreen.sh
```
#!/bin/bash

xrandr --newmode "1920x1080_60.00"  173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync
xrandr --addmode VGA-1 "1920x1080_60.00"
xrandr --output VGA-1 --mode "1920x1080_60.00"
```

- 赋予脚本可执行权限
>chmod +x /opt/script/myscreen.sh

---

## 定时检测网络状况
发现重启网络后网卡有起不来的情况，只能定时检测网络状态并重启网卡了。

- 编辑任务脚本
在/etc/cron.d/下存放要执行的crontab文件或脚本并赋予可执行权限（如/etc/cron.d/detectNetwork.sh）。
```
#!/bin/bash

function test()
{
    timeout=5
    target=www.baidu.com
    ret_code=`curl -I -s --connect-timeout $timeout $target -w %{http_code} | tail -n1`

    if [ "x$ret_code" != "x200" ]
    then
#        service network restart
        ifup ens33
    fi
    
    service teamviewerd restart
    echo `date` ': ok' >> /root/log.txt

}

test
```

- 添加任务(每4小时检查一次网络状况)
编辑/etc/crontab文件，在末尾添加如下内容
`0 */4 * * * root sh /etc/cron.d/detectNetwork.sh`
- crontab命令添加到系统
`crontab /etc/crontab`
- 查看crontab列表
`crontab -l`

## 参考
1. [CentOS域名解析慢的处理方法](https://it.oyksoft.com/post/6126/)
2. [shell脚本检测网络是否畅通](https://blog.csdn.net/wzy_1988/article/details/11634519)
3. [利用 xrandr 命令修改屏幕分辨率](https://www.jianshu.com/p/08c127007831)
4. [CentOS 7添加开机启动服务/脚本](https://blog.csdn.net/wang123459/article/details/79063703)
5. [Linux之crontab定时任务](https://www.jianshu.com/p/838db0269fd0)
6. [Linux---CentOS 定时运行脚本配置](https://blog.csdn.net/netdxy/article/details/50562864)
7. [命令行启动 TeamViewer](https://williamlfang.github.io/post/2017-12-05-%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%90%AF%E5%8A%A8-teamviewer/)