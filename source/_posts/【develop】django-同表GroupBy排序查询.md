---
title: 【develop】django 同表GroupBy排序查询
date: 2018-01-09 17:08:37
categories: develop
tags:
    - django
    - sql
---

通常django中涉及group by查询多是多个表之间的联合查询，使用外键搭建桥梁。碰到过同表group by查询的案例，发现弄起来还挺麻烦的，故记录下，涨涨见识。
<!-- more-->

## django annotate

django annotate注解功能可以用在多个表的分组统计，也可以用在同表的分组统计。 如果需要统计某个字段出现的数量需要使用'values'.

If you just need the total number of events for a single area, you don't need either annotate or aggregate, a simple count will do: 
`SomeModel.objects.filter(some_colname=xx_colname).count()`

If you want the count of events for multiple areas, you need annotate in conjunction with values: 
`SomeModel.objects.values('some_colname').annotate(num_col=Count('some_colname'))`

也就是说，如果要用某个字段出现的次数排序的话，通常这样使用： 
`SomeModel.objects.values('some_colname').annotate(num_col=Count('some_colname')).order_by('-num_col')`, 
这样做是没有问题的，但是返回的字段中就只有`some_colname`字段了，不能做到返回所有字段，或任意指定的字段。如果要满足这个条件，目前使用的是原生的sql语句，参考: [https://stackoverflow.com/questions/2283305/order-by-count-per-value](https://stackoverflow.com/questions/2283305/order-by-count-per-value)

```python
sql = ('select a.* from some_table a join(select xx_id, count(*) as cnt from some_table '
       'group by xx_id)b on (b.xx_id=a.xx_id) order by b.cnt desc')
records = SomeModel.objects.raw(sql)
```