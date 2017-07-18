---
title: sqlmap源码解析之test和boundary组合生成payload
date: 2017-03-30 17:59:43
categories: Web安全
tags:
    - sqlmap
---

## test和boundary组合生成payload

这部分主要梳理sqlmap生成payload的方式，sqlmap扫描规则文件位于`xml`文件夹中的`boundaries.xml`文件和`payloads`文件夹，payloads文件夹中存放了六种注入类型的xml文件。

* boolean_blind
* error_based
* inline_query
* stacked_queries
* time_blind
* union_query

最终payload的生成方式为: `prefix` + `payload` + `comment` + `suffix`, 其中`prefix`和`suffix`由boundaries中的子节点`prefix`和`suffix`提供，`payload`和`comment`由payloads文件夹中的test的子节点`payload`和`comment`提供。

<!-- more -->

## 解析test

test节点的格式如下：

```xml
<test>
    <title></title>
    <stype></stype>
    <level></level>
    <risk></risk>
    <clause></clause>
    <where></where>
    <vector></vector>
    <request>
        <payload></payload>
        <comment></comment>
        <char></char>
        <columns></columns>
    </request>
    <response>
        <comparison></comparison>
        <grep></grep>
        <time></time>
        <union></union>
    </response>
    <details>
        <dbms></dbms>
        <dbms_version></dbms_version>
        <os></os>
    </details>
</test>
```

### title

当前测试Payload的标题，通过标题可以了解当前的注入类型以及测试的数据库类型

### stype

SQL注入的类型, 可能的值和代表的意义如下

* 1: Boolean-based blind SQL injection
* 2: Error-based queries SQL injection
* 3: Inline queries SQL injection
* 4: Stacked queries SQL injection
* 5: Time-based blind SQL injection
* 6: UNION query SQL injection

### level

测试SQL注入的深度，共有5个级别，级别越高发送的请求数越多。这和boundary子节点level的含义是一致的

 * 1: Always (<100 requests)
 * 2: Try a bit harder (100-200 requests)
 * 3: Good number of requests (200-500 requests)
 * 4: Extensive test (500-1000 requests)
 * 5: You have plenty of time (>1000 requests)

### risk

测试payload破坏数据完整性的可能性。

* 1: Low risk
* 2: Medium risk
* 3: High risk

### clause[多个值用`,`分隔]

payload在哪个字段位置生效, 这和boundary子节点clause的含义是一致的

* 0: Always
* 1: WHERE / HAVING
* 2: GROUP BY
* 3: ORDER BY
* 4: LIMIT
* 5: OFFSET
* 6: TOP
* 7: Table name
* 8: Column name
* 9: Pre-WHERE (non-query)

### where[多个值用`,`分隔]

添加完整payload`<prefix> <payload><comment> <suffix>`的地方, 这和boundary子节点where的含义是一致的

 * 1: Append the string to the parameter original value
 * 2: Replace the parameter original value with a negative random integer value and append our string
 * 3: Replace the parameter original value with our string

### vector

注入向量

### request

注入测试发送的请求

* payload

    测试使用的payload, 其中的`[RANDNUM]`，[`DELIMITER_STAR]`，`[DELIMITER_STOP]`分别代表着随机数值与字符, 扫描时会用对应的随机数替换掉。

* comment

	添加在payload后面的注释

* char

   在union注入测试中暴力破解列数所使用的字符

* columns

    在union注入测试中测试列的范围， 如`<columns>[COLSTART]-[COLSTOP]</columns>`


### response

从响应中识别是否成功注入

* comparison(布尔盲注)

    应用比较算法，比较两次请求响应的不同，一次请求的payload使用request子节点中的`payload`, 另一次请求的payload使用response子节点中的`comparison`

* grep(报错注入)

    响应中匹配的正则表达式

* time(时间盲注和堆叠注入)

    响应返回之前等待的时间(单位为秒)

* union(union注入)

    调用 unionTest()函数

### details

注入成功能得到的关于操作系统和数据库相关的信息

* dbms

使用的数据库类型

* dbms_version

数据库版本

* os

运行数据库得操作系统

## 解析boundaries

boundary节点的格式如下:

```xml
<boundary>
    <level></level>
    <clause></clause>
    <where></where>
    <ptype></ptype>
    <prefix></prefix>
    <suffix></suffix>
</boundary>
```

其中`level`, `clause`, `where`表示的含义和test节点中所表示的含义是一致的。

### ptype

测试参数的类型

* 1: Unescaped numeric
* 2: Single quoted string
* 3: LIKE single quoted string
* 4: Double quoted string
* 5: LIKE double quoted string

### prefix

`<payload> <comment>`添加的前缀

### suffix

`<payload> <comment>`添加的后缀

## 源码解析payload组合

sqlmap使用`lib/controller/checks.py`文件中的`checkSqlInjection()`函数进行sql注入的测试, 其中利用test和boundary生成payload的主要代码如下，忽略其他逻辑判断

```python
# Favoring non-string specific boundaries in case of digit-like parameter values
if value.isdigit():
    kb.cache.intBoundaries = kb.cache.intBoundaries or sorted(copy.deepcopy(conf.boundaries), key=lambda boundary: any(_ in (boundary.prefix or "") or _ in (boundary.suffix or "") for _ in ('"', '\'')))
    boundaries = kb.cache.intBoundaries
else:
    boundaries = conf.boundaries

tests = getSortedInjectionTests()

while tests:
    test = tests.pop(0)

    for boundary in boundaries:
        injectable = False

        # Skip boundary if it does not match against test's <clause>
        # Parse test's <clause> and boundary's <clause>
        clauseMatch = False

        for clauseTest in test.clause:
            if clauseTest in boundary.clause:
                clauseMatch = True
                break

        if test.clause != [0] and boundary.clause != [0] and not clauseMatch:
            continue

        # Skip boundary if it does not match against test's <where>
        # Parse test's <where> and boundary's <where>
        whereMatch = False

        for where in test.where:
            if where in boundary.where:
                whereMatch = True
                break

        if not whereMatch:
            continue

        # For each test's <where>
        for where in test.where:
            # generage payload
            templatePayload = agent.payload(..., where=where)
```

利用test和boundary生成payload的流程为:

* 循环遍历每一个test,
* 对某个test,循环遍历boundary
* 若boundary的where包含test的where值，并且boundary的clause包含test的clause值, 则boundary和test可以匹配
* 循环test的where值,结合匹配的boundary生成相应的payload
