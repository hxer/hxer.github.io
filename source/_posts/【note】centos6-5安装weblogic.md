---
title: 【note】centos6.5安装weblogic
date: 2018-07-22 14:47:33
categories: note
tags: 
---


## 下载bin安装包

weblogic 10.3.6 linux可执行文件下载地址

* http://download.oracle.com/otn/linux/middleware/11g/wls/1036/wls1036_linux32.bin
<!-- more-->

## 安装

首先建立weblogic用户

```shell
groupadd -g 900 weblogic
useradd -u 900 -g weblogic weblogic
mkdir /usr/local/weblogic
chown -R weblogic.weblogic /usr/local/weblogic/
yum install -y glibc.i686 libstdc++.so.6
```

* 将本地主机名解析为127.0.0.1

否则会出现Weblogic：Could not obtain the localhost address问题

```
echo "127.0.0.1 $(hostname)" >> /etc/hosts

# 安装操作都以weblogic用户进行
su - weblogic
```

* 安装weblogic

参考: [CentOS 6通过bin安装包安装Weblogic](http://blog.51cto.com/linux10000/1957635)

安装路径为: /usr/local/weblogic

* 免密启动

安装完后先运行startWebLogic.sh脚本，会在base_domin目录下生成servers目录，然后设置免密登陆，假设用户名为weblogic, 密码为weblogic123

```shell
cd /usr/local/weblogic/user_projects/domains/base_domain/servers/AdminServer/
mkdir security
vim security/boot.properties
	username=weblogic
	password=weblogic123
```

重启weblogic

* 设置自启动

```shell
# 设置日志文件路径
touch /var/log/weblogic.log
chown weblogic:weblogic /var/log/weblogic.log
vim /etc/init.d/weblogic
```

```shell
#!/bin/bash
#chkconfig:35 99 05
#description:Weblogic Server

#/ect/init.d/weblogic
#Please edit the Variable
export WLS_HOME=/usr/local/weblogic/user_projects/domains/base_domain
export WLS_LOG=/var/log/weblogic.log
export PATH=$PATH:$WLS_HOME/bin
WLS_OWNER="weblogic"

if [ ! -f$WLS_HOME/bin/startWebLogic.sh -o ! -d $WLS_HOME ]
then
    echo "WebLogic startup:cannot start"
    exit 1
fi

# depending on parameter -- startup,shutdown,restart
case "$1" in
start)
    echo -n "Starting Weblogic:log file $WLS_LOG"
    touch /var/lock/weblogic
    su - $WLS_OWNER -c "nohup sh $WLS_HOME/bin/startWebLogic.sh > $WLS_LOG 2>$1 &"
    echo " OK"
    ;;
stop)
    echo -n "Shutdown Weblogic:"
    rm -rf /var/lock/weblogic
su - $WLS_OWNER -c "sh $WLS_HOME/bin/stopWebLogic.sh >> $WLS_LOG"
# killall -9 java
    echo " OK"
    ;;
reload|restart)
    $0 stop
    $0 start
    ;;
*)
    echo "Usage: `basename $0` start|stop|restart|reload"
    exit 1
esac
exit 0
```

```shell
chmod a+x/etc/init.d/weblogic

# 测试服务方式启动
service weblogic start
# 配置服务自启动
chkconfig weblogic on
```

## 踩过的坑

编写服务文件，调用weblogic启动脚本要放到后台执行，否则会阻塞之后的服务启动，比如ssh服务。

```shell
# 错误的方式，服务调用放在前台执行
su - ${WLS_OWNER} -c "cd ${WLS_HOME}; ./startWebLogic.sh"
# 正确的方式，服务调用放在后台执行
su - ${WLS_OWNER} -c "cd ${WLS_HOME}; ./startWebLogic.sh &"
```