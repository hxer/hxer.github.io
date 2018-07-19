---
title: 【note】clion远程调试
date: 2018-07-19 21:43:58
categories: note
tags:
---

mac上使用CLION远程调试centos上的C或C++可执行程序，clion使用GDB Remote Debug进行配置。

* 宿主机： OSX 10.13.6
* 虚拟机：CentOS 6.5 x64

## 虚拟机基本配置

首先在CentOS上安装必备的软件，包括 gcc, gcc-c++, gdb gdbserver, cmake, make。其中gcc-c++是用来编译cpp, cmake是由于clion使用cmake进行构建，因此CentOS上也要使用cmake来进行构建，保证本地和虚拟机的文件一致，在clion上使用GDB Remote Debug 进行远程调试。

```
yum install -y gcc gcc-c++ make cmake gdb gdb-gdbserve
```

<!--more-->
### cmake 构建

假设代码的根目录：/root/work/hello， 源代码为单文件main.cpp

项目目录下编写CMakeLists.txt文件

```
# Make 最低版本号要求
cmake_minimum_required(VERSION 2.8)

# 项目信息
project(hello)

# 指定源文件
set(SOURCE_FILES main.cpp)

set(CMAKE_SOURCE_DIR .)

# 配置gdb调试
set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g3 -ggdb")
set(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall")

# 指定生成目标
add_executable(hello ${SOURCE_FILES})
```

使用cmake构建

```
cd /root/work/hello
mkdir build
cd build
cmake .. 
make
```

在根目录下生成build目录，存放cmake构建的结果，其中包含生成的Makefile文件，然后使用make命令，会在build目录中生成可执行文件hello。

运行gdbserver

```
gdbserver 0.0.0.0:1234 /root/work/hello/build/hello
```

## Mac 远程连接调试

将虚拟机上的构建完成的整个项目文件拷贝到Mac上，存放路径为/Users/xxxuser/Pictures/hello

使用clion打开，配置debug选项为GDB Remote Debug。

```
GDB: Bundled GDB multiarch
'target remote' args: tcp:172.16.40.130:1234
Symbol file: /Users/xxxuser/Pictures/hello/build/hello
Path mapping: /root/work/hello /Users/xxxuser/Pictures/hello
```

'target remote' 参数中 172.16.40.130 为虚拟机的ip地址。

然后点击调试按钮，若连接上gdbserver, Console会显示 Debugger connected to tcp:172.16.40.130:1234。gdbserver 也会显示相应的连接信息。之后就可以愉快的使用clion进行远程调试了。

## 踩坑之旅

上面的步骤基本可以实现正常调试的功能。当然为了达成这个目标也是踩了不少坑，记录一下，备忘或给他人提供一些参考。

### clion gdb 连不上。

clion进行远程调试，出现连接不上的问题，结果如下

```
tcp:172.16.40.130:1234: Operation timed out.
Debugger disconnected
```

在mac上可以ssh登录虚拟机，但是clion死活连不上虚拟机的gdbserver。排除clion软件的代理等问题，首先在虚拟机上使用gdb进行远程连接

```
$gdb
(gdb) target remote 127.0.0.1:1234
(gdb) q

$gdb
(gdb) target remote 172.16.40.130:1234
(gdb) q
```

在虚拟机上使用gdb连接127.0.0.1和172.16.40.130两种方式进行连接，发现可以连上127.0.0.1，不能连上172.16.40.130，那么很可能是iptables的问题，禁用iptables后连接正常。

配置CentOS关闭防火墙

```
service iptables stop
chkconfig iptables off 
```

### No source file named main.cpp 问题

* 第一种 CMake 源文件路径配置

当CMakeLists.txt中未配置源文件路径时，clion启动gdb时，Debugger->GDB页面会出现No source file named main.cpp。

例如之前的错误配置为：

```
# Make 最低版本号要求
cmake_minimum_required(VERSION 2.8)

# 项目信息
project(hello)

# 配置gdb调试
set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g3 -ggdb")
set(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall")

# 指定生成目标
add_executable(hello main.cpp)
```

在CMakeLists.txt文件中添加`set(SOURCE_FILES main.cpp)`指定源文件，解决读取不到main.cpp的问题

* 第二种 程序编译选项

clion正常连接gdbserver后，在main.cpp文件中设置断点，出现No source file named main.cpp, 并且程断点也无效。

对应的cmake文件配置为`set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g -ggdb")`,使用的是-g选项, 经过网上一番搜索，设置-g3选项生效，即`set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g3 -ggdb")`。其中-g对应的是-g2, -g3产生更多的调试信息，例如-g3级别支持宏扩展。

可能是clion远程调试需要更多的调试信息，-g2级别不够，设置成-g3后就可以正常下断点调试，和本地调试无异。


