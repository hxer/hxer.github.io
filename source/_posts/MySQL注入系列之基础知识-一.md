---
title: MySQL注入系列之基础知识(一)
date: 2017-03-19 22:35:59
categories: Web安全
tags:
    - MySQL注入
---


## select 语法

```
SELECT
[ALL | DISTINCT | DISTINCTROW ]
  [HIGH_PRIORITY]
  [STRAIGHT_JOIN]
  [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
  [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
select_expr [, select_expr ...]
[FROM table_references
[WHERE where_condition]
[GROUP BY {col_name | expr | position}
  [ASC | DESC], ... [WITH ROLLUP]]
[HAVING where_condition]
[ORDER BY {col_name | expr | position}
  [ASC | DESC], ...]
[LIMIT {[offset,] row_count | row_count OFFSET offset}]
[PROCEDURE procedure_name(argument_list)]
[INTO OUTFILE 'file_name' export_options
  | INTO DUMPFILE 'file_name'
  | INTO var_name [, var_name]]
[FOR UPDATE | LOCK IN SHARE MODE]]
```

<!-- more -->

## 基础函数

[doc mysql 5.7 string-functions](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html)

| 函数 | 功能 | 备注 |
| ---- | ---  | ---- |
| concat | 返回连接参数产生的字符串 | 如果其中一个参数为null，则返回值为null |
| concat_ws | 返回连接参数产生的字符串 |CONCAT_WS(sp,s1,...)<br/> 第一个参数是分隔符，剩下的参数就是字段名 |
| group_concat | 合并多条记录中的结果 |  |
| if | |  if(exp1,exp2,exp2)<br/> exp1的表达式是True，返回exp2；否则返回exp3 |
| case | | |
| exists | | |
| length | 返回字符串的长度 |  |
| substr | 截断字符串  | substr(str,pos,length)<br/> pos是从1开始 |
| mid |  | mid(string, start, length) |
| left  | | left(string, n) |
| ord | 返回字符的ASCII码 | |
| ascii | 返回字符的ascii值 | |
| char |  | |
| hex | 返回字符串的16进制 |  |
| unhex | 返回字符串 |  |
| abs | |  |


### if

> if(expr1, expr2, expr2)

[mysql 5.7 doc](https://dev.mysql.com/doc/refman/5.7/en/control-flow-functions.html): IF() returns a numeric or string value, depending on the context in which it is used. IF() 函数根据其上下文决定返回 数字 或 字符串 类型的数据.

如果 expr2 或 expr3 中有一个类型为 NULL, 那么 IF() 函数的返回类型是另外一个非 NULL 表达式的类型.

IF() 默认返回值类型评定规则:

| 表达式 | 返回值类型 |
| --- | --- |
| expr2 或 expr3 返回字符串 | 字符串 |
| expr2 或 expr3 返回浮点数 | 浮点数 |
| expr2 或 expr3 返回整型 | 整型 |

当 expr2 或 expr3 都是字符串时, 如果他们之中有一个是字符大小写敏感的, 那么返回类型也是大小写敏感的

## mysql 常用过滤函数

## mysql安全配置
