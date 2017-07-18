---
title: CTF之MD5截断比较
date: 2017-03-21 16:23:39
categories: CTF
tags:
    - md5
---

## md5 截断验证

ctf中经常用MD5的截断比较做验证，如:`substr(md5($str), 0, 6) === "3322cf"`,通过这种方式限制脚本的自动化攻击。

通常可以写脚本去爆破MD5, 但是花费时间和比较字符串的长度有关，并且花费时间通常比较长，这不利于脚本自动攻击，下面给出爆破脚本和使用方式。

<!-- more -->

* submd5.py

```python
# -*- coding: utf-8 -*-

import multiprocessing
import hashlib
import random
import string
import sys


CHARS = string.letters + string.digits


def cmp_md5(substr, stop_event, str_len, start=0, size=20):
    global CHARS

    while not stop_event.is_set():
        rnds = ''.join(random.choice(CHARS) for _ in range(size))
        md5 = hashlib.md5(rnds)

        if md5.hexdigest()[start: start+str_len] == substr:
            print rnds
            stop_event.set()


if __name__ == '__main__':
    substr = sys.argv[1].strip()

    start_pos = int(sys.argv[2]) if len(sys.argv) > 1 else 0

    str_len = len(substr)
    cpus = multiprocessing.cpu_count()
    stop_event = multiprocessing.Event()
    processes = [multiprocessing.Process(target=cmp_md5, args=(substr,
                                         stop_event, str_len, start_pos))
                 for i in range(cpus)]

    for p in processes:
        p.start()

    for p in processes:
        p.join()
```

用法:

```
$ python submd5.py "3d4f4"
SponhjOhIZ30IaM1fweb

$ python submd5.py "3df4" 2
G8tr6VhonA1z3xJdaGBu
```

想要在较短时间内获得可用的md5,可以使用彩虹表类似的方式去实现，通过空间去换时间。

生成md5文件，并排序:

* gen_md5.py

```python
# -*- coding: utf-8 -*-

import itertools
import hashlib
import string


CHARS = string.letters + string.digits
str_len = 8

for s in itertools.product(CHARS, repeat=str_len):
    s = ''.join(s)
    print "{0} {1}".format(hashlib.md5(s).hexdigest(), s)
```

命令行排序: `python gen_md5.py | sort -o md5_sorted.txt`

md5 文件搜索(二分查找):

* match.py

```python
# -*- coding: utf-8 -*-

import itertools
import hashlib
import string
import os


def match(s):
    md5_file = "md5_sorted.txt"
    byte_size = os.path.getsize(md5_file)
    with open(md5_file, 'rb') as f:
        line_len = len(f.readline())

    print line_len
    with open(md5_file, "rb") as f:
      L = 0
      R = byte_size / line_len - 1

      while R - L > 0:
        C = L + (R - L) / 2
        offset = C * line_len
        f.seek(offset)
        ln = f.read(line_len).strip()
        #print ln

        head = ln[:len(s)]
        if s == head:
          return ln.split(" ")[1]

        if s < head:
          R = C
          continue

        L = C

      return

# print match('fef')
```
