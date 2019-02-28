---
title: 【pwn】IDA远程调试方法
date: 2019-02-22 14:04:45
categories: pwn
tags:
    - IDA Pro
---


### 环境准备

* 宿主机
  * 操作系统：OSX 10.13
  * IDA PRO: 7.0
* 虚拟机
  * 操作系统: ubuntu 16.04

1. 拷贝IDA文件到虚拟机中，

   `/Applications/IDA Pro 7.0/ida.app/Contents/MacOS/dbgsrv/linux_server64`

   拷贝`linux_server64`到虚拟机，则宿主机打开IDA64版本，要求版本一致。

2. 虚拟机执行监听

   ```shell
   #/home/xxx/Desktop/linux_server64
   $chmod +x /home/xxx/Desktop/linux_server64
   
   # 待分析的二进制文件 /home/xxx/Desktop/pwne
   $chmod +x /home/xxx/Desktop/pwne
   ```
<!-- more -->
3. 打开IDA64
    {% asset_img ida01.png %}
   选择Debugger->run->Remote Linux debugger
   {% asset_img ida02.png %}
