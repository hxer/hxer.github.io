---
title: 【note】CentOS免密登录问题排查
date: 2018-07-19 21:43:44
categories: note
tags:
---

CentOS 6.5 免密登陆问题排查，服务器ssh服务器运行正常，使用用户名和密码登陆服务器正常，但是使用秘钥登陆一直失败，于是先排查sshd配置文件`/etc/ssh/sshd_config`,支持秘钥认证登陆

```
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
```

修改文件重启ssh服务后，仍旧不能免密登陆。于是排查各文件和文件夹权限，修改为

```
drwx------. 3 root root 4096 7月  18 20:22 /root
drwx------.  2 root root 4096 7月  18 22:56 .ssh
-rw-------. 1 root root  396 7月  18 22:56 authorized_keys
```

还是不行，从客户端`ssh -v`显示的信息依旧看不出什么问题，接下来排查ssh服务器日志。

<!--more-->
## ssh 服务器日志排查

ssh服务器日志位于 `/var/log/secure`，默认日志记录的INFO信息，需要修改日志记录级别为DEBUG，修改`/etc/ssh/sshd_config`文件中`LogLevel INFO` 为 `LogLevel DEBUG`，重启ssh服务器`service sshd restart`。

尝试ssh登陆，发现问题在于不能读取authorized_keys

```
Jul 18 23:01:05 centos sshd[1564]: debug1: trying public key file /root/.ssh/authorized_keys
Jul 18 23:01:05 centos sshd[1564]: debug1: Could not open authorized keys '/root/.ssh/authorized_keys': Permission denied
```

参考 https://stackoverflow.com/questions/20688844/sshd-gives-error-could-not-open-authorized-keys-although-permissions-seem-corre，问题在于SELinux阻止sshd打开文件。

执行：`restorecon -FRvv ~/.ssh`， 解决问题。

## 问题分析

该问题本质应该是`authorized_keys`文件的context不正确，正常情况下应该是`ssh_home_t`，如果是`admin_home_t`则是错误的。可以使用`estorecon`命令进行恢复。

```
[root@centos ~]# ls -alZ .ssh/
drwx------. root root system_u:object_r:ssh_home_t:s0  .
drwx------. root root system_u:object_r:admin_home_t:s0 ..
-rw-------. root root system_u:object_r:ssh_home_t:s0  authorized_keys
```


