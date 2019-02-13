---
title: 【note】ubuntu16安装jenkins
date: 2018-08-13 11:15:39
categories: note
tags:
    - linux
    - jenkins
---

## 环境搭建

```
sudo add-apt-repository ppa:openjdk-r/ppa
sudo apt-get update

sudo apt-get install openjdk-8-jdk
sudo apt-get install git
```

### 设置java环境变量

vim ~/.bashrc

```
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

<!--more -->

### 安装指定版本maven

> maven版本要求为3.5以上

* 下载二进制包：

[maven 3.5.4](http://mirror.bit.edu.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz)

* 解压：

```
tar xzvf apache-maven-3.5.4-bin.tar.gz
````

* 设置maven 环境变量

```
vim ~/.bashrc

export M2_HOME=/home/xxxUser/apache-maven-3.5.4
export M2=$M2_HOME/bin
export PATH=$M2:$PATH
```

## 编译和调试jenkins

````
git clone https://github.com/jenkinsci/jenkins.git
git checkout jenkins-2.121
```

```
# 编译，跳过单元测试
$ mvn clean install -pl war -am -DskipTests

# 编译完成 在war/target目录下找到jenkins.war了，运行如下命令Jenkins就启动起来了。
# 访问8080端口, 完成安装
$java -jar jenkins.war 

# 调试，这样跑起来的 jenkins 实例，会额外监听 8000 端口(jdwp端口)，用 IDE 提供的 remote debug 功能连上去
$ cd war
$ mvnDebug jenkins-dev:run
```



### 备注

不要修改maven镜像源，可能会出现一些依赖包下载不了的情况。

修改镜像源的方式为：

```
$ cd  $M2_HOME/conf/
$ sudo vim settings.xml

<mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus aliyun</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror>
````
