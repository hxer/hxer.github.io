---
title: 【data】入门Flink scala编程
date: 2019-01-14 16:45:10
categories: data
tags:
    - flink
    - scala
---

flink scala编程入门的教程网上有一些，看着步骤相对比较简单，但是实践起来还是遇到不少的坑，这里记录下过程，供日后查阅，备注附录了遇到的一些问题。至于代码解释可自行查阅网上其他资料，这里不再赘述。

<!--more -->

## 工程构建

* IDEA新建一个空的maven项目，`src/main`目录下新建scala文件夹，并配置为`Sources`类型

* 配置pom.xml文件

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
      <modelVersion>4.0.0</modelVersion>
  
      <groupId>com.eflink</groupId>
      <artifactId>word-count</artifactId>
      <version>1.0</version>
  
      <properties>
          <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
          <flink.version>1.7.1</flink.version>
          <scala.binary.version>2.11</scala.binary.version>
          <scala.version>2.11.12</scala.version>
      </properties>
  
      <dependencies>
          <!-- Apache Flink dependencies -->
          <!-- These dependencies are provided, because they should not be packaged into the JAR file. -->
          <dependency>
              <groupId>org.apache.flink</groupId>
              <artifactId>flink-scala_${scala.binary.version}</artifactId>
              <version>${flink.version}</version>
              <scope>provided</scope>
          </dependency>
  
          <dependency>
              <groupId>org.apache.flink</groupId>
              <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
              <version>${flink.version}</version>
              <scope>provided</scope>
          </dependency>
  
          <dependency>
              <groupId>org.apache.flink</groupId>
              <artifactId>flink-clients_${scala.binary.version}</artifactId>
              <version>${flink.version}</version>
              <scope>provided</scope>
          </dependency>
  
          <dependency>
              <groupId>org.scala-lang</groupId>
              <artifactId>scala-library</artifactId>
              <version>${scala.version}</version>
          </dependency>
      </dependencies>
  
      <build>
          <finalName>word-count</finalName>
          <plugins>
              <plugin>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-shade-plugin</artifactId>
                  <version>3.0.0</version>
                  <executions>
                      <!-- Run shade goal on package phase -->
                      <execution>
                          <phase>package</phase>
                          <goals>
                              <goal>shade</goal>
                          </goals>
                          <configuration>
                              <artifactSet>
                                  <excludes>
                                      <exclude>org.apache.flink:force-shading</exclude>
                                      <exclude>com.google.code.findbugs:jsr305</exclude>
                                      <exclude>org.slf4j:*</exclude>
                                      <exclude>log4j:*</exclude>
                                  </excludes>
                              </artifactSet>
                              <filters>
                                  <filter>
                                      <!-- Do not copy the signatures in the META-INF folder.
                                      Otherwise, this might cause SecurityExceptions when using the JAR. -->
                                      <artifact>*:*</artifact>
                                      <excludes>
                                          <exclude>META-INF/*.SF</exclude>
                                          <exclude>META-INF/*.DSA</exclude>
                                          <exclude>META-INF/*.RSA</exclude>
                                      </excludes>
                                  </filter>
                              </filters>
                              <transformers>
                                  <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                      <mainClass>com.eflink.SocketWordCount</mainClass>
                                  </transformer>
                              </transformers>
                          </configuration>
                      </execution>
                  </executions>
              </plugin>
  
              <!-- Scala Compiler -->
              <plugin>
                  <groupId>net.alchim31.maven</groupId>
                  <artifactId>scala-maven-plugin</artifactId>
                  <version>3.4.4</version>
                  <executions>
                      <execution>
                          <goals>
                              <goal>compile</goal>
                              <goal>testCompile</goal>
                          </goals>
                      </execution>
                  </executions>
              </plugin>
  
          </plugins>
      </build>
  </project>
  ```

  pom.xml文件中设置flink和scala的版本,本机flink版本为`1.7.1`，scala的版本根据flink的版本来确定，例如：依赖[Flink Clients](https://mvnrepository.com/artifact/org.apache.flink/flink-clients)，查看特定flink版本下scala版本支持的情况。


### 编写代码

参考[Flink tutorials scala code](https://ci.apache.org/projects/flink/flink-docs-release-1.7/tutorials/local_setup.html),IDEA创建scala class选择Object，代码如下:

```scala
package com.eflink

import org.apache.flink.api.java.utils.ParameterTool
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.scala._

object SocketWordCount {
  def main(args: Array[String]): Unit = {
    // the port to connect to
    val port: Int = try {
      ParameterTool.fromArgs(args).getInt("port")
    } catch {
      case e: Exception => {
        System.err.println("No port specified. Please run 'SocketWindowWordCount --port <port>'")
        return
      }
    }

    // get the execution environment
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

    // get input data by connecting to the socket
    val text = env.socketTextStream("localhost", port, '\n')

    // parse the data, group it, window it, and aggregate the counts
    val windowCounts = text
      .flatMap { w => w.split("\\s") }
      .map { w => WordWithCount(w, 1) }
      .keyBy("word")
      .timeWindow(Time.seconds(5), Time.seconds(1))
      .sum("count")

    // print the results with a single thread, rather than in parallel
    windowCounts.print().setParallelism(1)

    env.execute("Socket Window WordCount")
  }

  // Data type for words with count
  case class WordWithCount(word: String, count: Long)
}
```

* `nc -l 9000`
* `Cellar/apache-flink/1.7.1/libexec/bin/start-cluster.sh`
* `mvn compile/package`
* `flink run target/word-count.jar --port 9000` 

## 备注

1. maven更换阿里云源

   ```shell
   cp /usr/local/Cellar/maven@3.3/3.3.9/libexec/conf/settings.xml ~/.m2
   
   vim ~/.m2/settings.xml
   
   <mirrors>
     <!-- mirror
      | Specifies a repository mirror site to use instead of a given repository. The repository that
      | this mirror serves has an ID that matches the mirrorOf element of this mirror. IDs are used
      | for inheritance and direct lookup purposes, and must be unique across the set of mirrors.
      |
     <mirror>
       <id>mirrorId</id>
       <mirrorOf>repositoryId</mirrorOf>
       <name>Human Readable Name for this Mirror.</name>
       <url>http://my.repository.com/repo/path</url>
     </mirror>
      -->
      <mirror>
       <id>alimaven</id>
       <name>aliyun maven</name>
       <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
       <mirrorOf>central</mirrorOf>       
     </mirror>
   </mirrors>
   ```

2. 包导入问题

   IDEA根据编写的代码能够自动或手动提示导入所需要的包，但是构建官方给的代码中会报如下错误：

   ```
   error: could not find implicit value for evidence parameter of type org.apache.flink.api.common.typeinfo.TypeInformation[String]
   [ERROR]       .flatMap { w => w.split("\\s") }
   ```

   这是因为程序需要一个隐形参数导致的，引用包改为`import org.apache.flink.streaming.api.scala._`

3. 单纯scala项目pom文件

   ```xml
   <dependency>
       <groupId>org.scala-lang</groupId>
       <artifactId>scala-library</artifactId>
       <version>${scala.version}</version>
   </dependency>
   
   <plugin>
       <groupId>net.alchim31.maven</groupId>
       <artifactId>scala-maven-plugin</artifactId>
       <version>3.4.4</version>
       <executions>
           <execution>
               <goals>
                   <goal>compile</goal>
                   <goal>testCompile</goal>
               </goals>
           </execution>
       </executions>
   </plugin>
   
   <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-assembly-plugin</artifactId>
       <version>2.5.5</version>
       <configuration>
           <archive>
               <manifest>
                   <mainClass>packagename.classname</mainClass>
               </manifest>
           </archive>
           <descriptorRefs>
               <descriptorRef>jar-with-dependencies</descriptorRef>
           </descriptorRefs>
       </configuration>
       <executions>
           <execution>
               <id>make-assembly</id>
               <phase>package</phase>
               <goals>
                   <goal>single</goal>
               </goals>
           </execution>
       </executions>
   </plugin>
   ```

4. 远程提交

   本地开发然后往集群提交远程作业，ExecutionEnvironment.createRemoteEnvironment 第三个参数是打包之后的jar包路径，也是必不可少，不然会找不到类，同时这个jar包是单一jar包，不是那种把所有依赖都打进去的可执行jar包。

5. IDEA配置scala SDK

    打开scala文件，IDEA会弹出通知`No Scala SDK in module   Setup Scala SDK`，需要配置scala sdk，点击`Setup Scala SDK`配置。

    1. **选择maven下载的scala**(推荐)
    2. 选择本地安装的scala

    查看本地安装scala的位置

    ```shell
    ▶ brew info scala
    scala: stable 2.12.8
    JVM-based programming language
    https://www.scala-lang.org/
    /usr/local/Cellar/scala/2.12.8 (42 files, 20.8MB) *
    Built from source on 2019-01-08 at 12:30:35
    From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/scala.rb
    ==> Requirements
    Required: java >= 1.8 ✔
    ```

    scala安装在`/usr/local/Cellar/scala/2.12.8`，但是IDEA弹出的finder不能选择`/usr`文件夹，于是曲线救国，创建一个软链接到可访问的目录下：

    ```shell
    ln -s /usr/local/Cellar/ /Users/xxxuser/Cellar
    ```

    于是可以通过当前用户目录下的Cellar软连接，选择`/usr/local/Cellar/scala/2.12.8/libexec`配置scala sdk。

6. scala版本问题

    flink版本为1.7.1，推荐使用scala2.11版本，虽说1.7.1的flink版本支持scala 2.11和2.12，但是在实践中发现，当使用scala 2.12版本时，运行程序会报错`java.lang.NoSuchMethodError: scala.Predef$.refArrayOps([Ljava/lang/Object;)[Ljava/lang/Object;`，将scala版本改为2.11后程序可正常运行。查阅网上资料，有说是jdk版本和scala版本适配问题导致的，目前未深入分析，具体原因不明。

7. 环境配置

    - 安装flink

        ```shell
        ▶ brew install apache-flink
            
        ▶ flink --version
        Version: 1.7.1, Commit ID: 89eafb4
        ```

    - 安装scala(非必需)

        ```shell
        ▶ brew install scala@2.11
        ```

    - 安装mvn(非必需)

        ```
        ▶ brew install maven@3.3
        ▶ brew info maven@3.3
        maven@3.3: stable 3.3.9 [keg-only]
        Java-based project management
        https://maven.apache.org/
        /usr/local/Cellar/maven@3.3/3.3.9 (91 files, 9.6MB)
        
        # echo 'export PATH="/usr/local/opt/maven@3.3/bin:$PATH"' >> ~/.zshrc
        # or
        ▶ ln -s /usr/local/Cellar/maven@3.3/3.3.9/bin/mvn /usr/local/bin/mvn
        
        ▶ mvn -v
        Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
        Maven home: /usr/local/Cellar/maven@3.3/3.3.9/libexec
        Java version: 1.8.0_191, vendor: Oracle Corporation
        Java home: /Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home/jre
        Default locale: zh_CN, platform encoding: UTF-8
        OS name: "mac os x", version: "10.13.6", arch: "x86_64", family: "mac"
        ```

    - IDEA 安装`scala`插件



### 参考

* [http://wuchong.me/categories/Flink/](http://wuchong.me/categories/Flink/)

* [pom.xml in flink scala demo](https://gist.github.com/hxer/e2d55d4e82b45b22f9127b7501941c63)

