---
title: requests手动修改Content-Length头
date: 2017-03-21 13:13:47
categories: Python
tags:
    - requests
---

## requests发送前修改请求信息

requests可以再发送请求前，修改请求信息，见[Prepared Requests](http://docs.python-requests.org/en/latest/user/advanced/#prepared-requests)

<!-- more -->

例如发送前手动修改请求头`Content-Length`

```python

from requests import Request, Session

s = Session()
req = Request('POST', url, data=data, headers=headers)
prepped = req.prepare()

# do something with prepped.headers
prepped.headers['Content-Length'] = your_custom_content_length_calculation()
# prepped.headers['Content-Length'] = '1000000000'

resp = s.send(prepped)
```

如果`session`有自己的配置，如cookie等，应该使用`s.prepare_request(req)`, 而不是`req.prepare()`
