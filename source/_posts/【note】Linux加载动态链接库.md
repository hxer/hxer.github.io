---
title: 【note】Linux加载动态链接库
date: 2017-12-19 23:45:19
categories: note
tags:
    - linux
    - 动态链接库
---

### LIBRARY_PATH 和 LD_LIBRARY_PATH

LIBRARY_PATH和LD_LIBRARY_PATH是Linux下的两个环境变量，二者的含义和作用分别如下：

1. LIBRARY_PATH环境变量用于在**\*程序编译期间***查找动态链接库时指定查找共享库的路径，例如，指定gcc编译需要用到的动态链接库的目录。
2. LD_LIBRARY_PATH环境变量用于在**\*程序加载运行期间***查找动态链接库时指定除了系统默认路径之外的其他路径，注意，LD_LIBRARY_PATH中指定的路径会在系统默认路径之前进行查找。

**区别与使用：**

1. 开发时，设置LIBRARY_PATH，以便gcc能够找到编译时需要的动态链接库。
2. 发布时，设置LD_LIBRARY_PATH，以便程序加载运行时能够自动找到需要的动态链接库。
3. GCC里的链接器的选项是 -rpath 和 -rpath-link
<!-- more-->
## 临时指定动态链接库

LD_LIBRARY_PATH 或 LD_PRELOAD

* 设置LD_LIBRARY_PATH

```
export LD_LIBRARY_PATH=/home/xxx/Desktop:$LD_LIBRARY_PATH
```

其中 **/home/plusls/Desktop** 为so文件所在的目录

**注：这样设置后 pwntools 起的进程也会继承该环境变量，加载此libc**

* 设置LD_PRELOAD

终端设置LD_PRELOAD，指定程序运行要加载的动态链接库，如：

```bash
LD_PRELOAD=./libc.so.6 ./app
```

