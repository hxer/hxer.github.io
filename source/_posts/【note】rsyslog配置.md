---
title: 【note】rsyslog配置
date: 2018-08-10 19:23:36
categories: note
tags:
    - linux
---

Linux日志机制的核心是rsyslog守护进程，该服务负责监听Linux下的日志信息，并把日志信息追加到对应的日志文件中，一般在/var/log目录下。它还可以把日志信息通过网络协议发送到另一台Linux服务器上。
<!--more-->

## 配置文件

rsyslog的配置文件为/etc/rsyslog.conf和/etc/rsyslog.d/*.conf。

### 服务器上存储的日志文件可以按天进行存储

示例配置如下：

* /etc/rsyslog.d/work.conf

```shell
# Create dynamic file template
$template DynLogs,"/var/log/app/%PROGRAMNAME%_%$YEAR%-%$MONTH%-%$DAY%.log" *
# save localhost
local5.alert  ?DynLogs
```

rsyslog 接收local5程序alert及其以上事件的日志，存放在/var/log/app路径，通过年月日命名文件的方式实现按日期存储日志。

### 指定日志信息格式

配置示例如下：

```shell
# Create dynamic file template
$template DynLogs,"/var/log/app/%PROGRAMNAME%_%$YEAR%-%$MONTH%-%$DAY%.log" *
$template GRAYLOGRFC5424,"<%PRI%>%PROTOCOL-VERSION% %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n"
# save localhost
local5.alert  ?DynLogs;GRAYLOGRFC5424
```

### 转发到远程服务器

配置示例如下：

```
# forwarding ymon log to remote server
$WorkDirectory /var/lib/rsyslog # where to place spool files
$ActionQueueFileName fwdWORK # unique name prefix for spool files
$ActionQueueMaxDiskSpace 4g   # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on # save messages to disk on shutdown
$ActionQueueType LinkedList   # run asynchronously
$ActionResumeRetryCount -1    # infinite retries if host is down
local5.alert @@172.16.40.12:514

& ~
```

远程服务器rsyslog配置,同样设置按天进行存储

```shell
#### MODULES #### 
# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514

#$AllowedSender tcp, 10.6.1.0/24
$template RemoteLogs,"/var/log/%HOSTNAME%/%PROGRAMNAME%_%$YEAR%-%$MONTH%-%$DAY%.log" *
local5.alert  ?RemoteLogs
& ~
```

若开启防火墙，则需配置

```shell
vim /etc/sysconfig/iptables 

# ssh 配置行下添加一行 ssh: -A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 514 -j ACCEPT

service iptables restart
```