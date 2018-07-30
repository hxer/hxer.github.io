---
title: 【note】linux上限制进程cpu占用率
date: 2018-07-30 20:37:13
categories: note
tags: 
    - linux
---

linux上限制进程cpu占用率，可以使用cgroups或者cpulimit，这两种方式经实践能有效限制cpu的占用。cgroups现已集成到大多数linux中，简单配置即可实现各种对进程资源的配置，优先推荐使用该方式，系统资源消耗少，且稳定可靠，配置方式灵活。cpulimit是一个开源工具，github地址为 [https://github.com/opsengine/cpulimit](https://github.com/opsengine/cpulimit)。cpulimit利用信号机制暂停和恢复进程的运行，并不直接操作内核。

<!-- more -->

## cgroups

cgroups 目的是对进程进行资源管理,涉及以下概念：

* task：任务，对应于系统中运行的一个实体，一般是指进程
* subsystem：子系统，具体的资源控制器（resource class 或者 resource controller），控制某个特定的资源使用。比如 CPU 子系统可以控制 CPU 时间，memory 子系统可以控制内存使用量
* cgroup：控制组，一组任务和子系统的关联关系，表示对这些任务进行怎样的资源管理策略
* hierarchy：层级树，一系列 cgroup 组成的树形结构。每个节点都是一个 cgroup，cgroup 可以有多个子节点，子节点默认会继承父节点的属性。系统中可以有多个 hierarchy

更多介绍参考: [redhat CGROUPS](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01)

以下使用实例方式，介绍使用cgroups区分用户限制进程对cpu的占用率。

* 系统：centos 6.5 
* cpu数：2
* 目标
    * test 用户进程cpu 100% （单cpu)
    * 其他任意用户进程cpu 60% (单cpu)

### 新建test用户

```
/usr/sbin/adduser test
passwd test

# test用户赋予sudo权限
/usr/sbin/visudo

# User privilege specification
root    ALL=(ALL)       ALL
test   ALL=(ALL)       ALL
```

### 配置cgroups

```
yum install -y libcgroup
service cgconfig start

# 查看子系统
ls /cgroup
blkio  cpu  cpuacct  cpuset  devices  freezer  memory  net_cls

lscgroup
cpuset:/
cpu:/
cpuacct:/
memory:/
devices:/
freezer:/
net_cls:/
blkio:/
```

* 配置cgconfig

```
vim /etc/cgconfig.conf

# append
group limitcpu {
	cpu {
		cpu.cfs_quota_us = 6000;
		cpu.cfs_period_us = 10000;
	}
}

group user {
	cpu {
		cpu.cfs_quota_us = 10000;
		cpu.cfs_period_us = 10000;
	}
}
```

* 配置cgrules

```
vim /etc/cgrules.conf

# append
test    cpu    user/
*       cpu    limitcpu/
```

* 服务启动和自启

```
service cgconfig restart
chkconfig cgconfig on
service cgred start
chkconfig cgred on
```

## cpulimit

从github下载源码，make编译，会在src目录生成可执行程序cpulimit.

对进程 xxx (pid:1234)限制cpu占用率为 30%，可以使用以下命令

```
./cpulimit -p 1234 -l 30
./cpulimit -e xxx -l 30
```

cpulimit原理比较简单，计算进程cpu的占用时间，然后调用`kill(pid,SIGSTOP)`和`kill(pid,SIGCONT)`来暂停和恢复进程，在此期间调用`nanosleep`后台挂起，避免cpulimit过多占用cpu时间。

## 参考

* [Linux 内核 cgroups 简介](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [How To Limit Resources Using cgroups on CentOS 6](https://www.digitalocean.com/community/tutorials/how-to-limit-resources-using-cgroups-on-centos-6)
