---
title: 【note】centos6 升级gcc和g++
date: 2018-08-03 13:31:31
categories: note
tags:
    - linux
---

对 gcc或g++ 升级无法直接使用：

```
yum update gcc
```

以下是升级的详细过程。

### 1.使用 redhat developer toolset 2 的repo，安装GCC

```
cd /etc/yum.repos.d
wget https://people.centos.org/tru/devtools-2/devtools-2.repo
yum --enablerepo=testing-2-devtools-6 install devtoolset-2-gcc devtoolset-2-gcc-c++
```

<!--more-->
### 2. 替换系统中原来的GCC

通过通过第一步会把 GCC 安装到以下目录：

```
/opt/rh/devtoolset-2/root/usr/bin/
```

接下来需要修改系统的配置，使默认的 gcc 和 g++ 命令使用的是新安装的版本。

```
ln -s /opt/rh/devtoolset-2/root/usr/bin/* /usr/local/bin/
hash -r
```

现在查看 g++ 的版本号：

```
g++ --version
g++ (GCC) 4.8.2 20140120 (Red Hat 4.8.2-15)
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

g++版本升级到4.8.2了。

### 3. 副作用

gcc 4.8生成“rep;ret”指令，避免AMD芯片的性能损失，而老版的assemblers会识别为错误，
解决方法是更新binutils，使用更新的assemblers接受此指令。参考：[https://gcc.gnu.org/ml/gcc-help/2011-03/msg00286.html](https://gcc.gnu.org/ml/gcc-help/2011-03/msg00286.html)

```shell
yum install -y devtoolset-2-binutils-devel
```


