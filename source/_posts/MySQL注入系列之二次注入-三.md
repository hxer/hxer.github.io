---
title: MySQL注入系列之二次注入(三)
date: 2017-03-28 13:33:04
categories: Web安全
tags:
    - MySQL注入
---

## 二次注入原理

二次注入漏洞字面上理解可能就是结合两个注入漏洞点实现sql注入的目的，但是这其中还有几个细节需要讲解一下。首先，第一个注入点因为经过过滤处理所以无法触发sql注入漏洞，比如`addslashes`函数，将单引号等字符转义变成\'。但是存进数据库后，数据又被还原了，也就是反斜杠没了，在这种情况下，如果能发现一个新的注入同时引用了被插入了的数据库数据，就可以实现闭合新发现的注入漏洞引发漏洞。

{% asset_img doubleinject.png %}

<!-- more -->

## 演示

程序提供用户注册(`reg.php`)和邮箱搜索(`search.php`)两个功能，程序在`config.php`文件中使用`addslashes`转义所有的GPC输入，然后再将数据插入数据库中，如下图所示，用户输入含有`'`的用户名会完整的存入数据库。

{% asset_img reg.png %}
{% asset_img database.png %}

`'`被安全的存入了数据库，如果又将该用户名从数据库中取出来，然后再用于数据库查询，若没转义该用户名，则会出现二次注入，演示如下:

搜索邮箱：`e@qq.com`, 得到mysql报错如下，显然`'`引发了数据库错误，也就是说这里是可以利用sql注入的。

{% asset_img error.png %}

这里是可以利用盲注，union注入和报错注入的, 当然，注入的利用还和username等字段的长度限制有关。

注册用户名: `' union select 1,user(),2,3 #`, 邮箱:`cc@qq.com`, 然后搜索邮箱`cc@qq.com`, 得到如下结果:

{% asset_img user.png %}

### 演示代码

* config.php

```php
<?php
mysql_connect('localhost', 'root', 'mysql');
mysql_select_db('sqlinject');
mysql_set_charset('utf-8');
if (!get_magic_quotes_gpc()){
    if (!empty($_GET)){
        $_GET = addslashes_deep($_GET);
    }
    if (!empty($_POST)){
        $_POST = addslashes_deep($_POST);
    }
    $_COOKIE = addslashes_deep($_COOKIE);
    $_REQUEST = addslashes_deep($_REQUEST);
}

function addslashes_deep($value){
    if (empty($value)){
        return $value;
    }else {
        return is_array($value) ? array_map('addslashes_deep', $value): addslashes($value);
    }
}
?>
```

* reg.php

```php
<?php
include "config.php";
if(!empty($_POST['submit'])){
    $username = $_POST['username'];
    $password = $_POST['password'];
    $email = $_POST['email'];

    $sql = "INSERT INTO `sqlinject`.`users` (`id`, `username`, `password`, `email`)
           VALUE (NULL, '$username', '$password', '$email');";
    $row = mysql_query($sql);
    if ($row){
        echo "注册成功";
    } else {
        echo "注册失败";
    }
}
?>

<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<form action="" method="POST">
    username<input type="text" name="username"><br/>
    password<input type="text" name="password"><br/>
    email<input type="text" name="email"><br/>
    <input type="submit" name="submit" value="ok">
</form>
```

* search.php

```php
<?php
include "config.php";
if(!empty($_POST['submit'])){
    $email = $_POST['email'];
    $sql = "select * from users where email='{$email}'";
    $row = mysql_query($sql);

    if ($row){
        $rows = mysql_fetch_array($row);
        $username = $rows['username'];
        $sql = "select * from users where username='$username'";
        $row = mysql_query($sql) or die(mysql_error());
        if ($row){
            $rows = mysql_fetch_array($row);
            echo $rows['username']."<br/>";
            echo $rows['password']."<br/>";
            echo $rows['email']."<br/>";
        }
    }
}
?>

<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<form action="" method="POST">
    search email<input type="text" name="email"><br/>
    <input type="submit" name="submit" value="ok">
</form>
```
