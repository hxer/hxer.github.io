---
title: selenium+phantomjs爬取新浪微博
date: 2017-03-14 19:05:11
categories: Python
tags:
    - 新浪微博
    - selenium
    - phantomjs
    - 爬虫
---

## 背景介绍

抓了twitter的一些数据，自然想到抓一下微博的数据，做个对比，比twitter麻烦的是，微博需要登录才能访问更多的记录，或者使用新浪微博的API,不过API的限制也是蛮多的，所以依旧还是优先考虑爬虫。

模拟登录新浪微博一般有以下几种方法：

* 用户cookie登录
* 账号密码登录，手动输入验证码
* 二维码扫描登录

以上三种方法各有优劣，考虑到现在的需求就只是抓取关键词搜索的结果，那么就采用最简便的cookie登录方式了。本想着用requests模拟登录，然后用beautifulsoup提取数据就行了的，发现返回的结果可读性很差，数据都放在script中，看着眼瞎，于是就想直接换`selenium+phantomjs`来爬取数据，这样可以直接用chrome定位数据，方便提取和调试。

<!-- more -->

## 问题记录

初始化phantomjs，设置指定的cookie参数，第一次访问，登录成功，然而第二次访问的时候，pohantomjs根据服务器响应的`Set-Cookie`把原始设定的请求Cookie给换掉了，从而导致登录失败。一种可行的方式就是，每次访问都指定cookie, phantomjs设置cookie有固定的格式，记录如下:

```python
for key, value in cookies.items():
    c = {}
    c['name'] = key
    c['value'] = value
    c['domain'] = '.weibo.com'
    c['path'] = '/'
    c['httponly'] = False
    c['secure'] = False
    self.driver.add_cookie(c)
```

## 实现

* weibo_search.py

```python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals
import time
from selenium import webdriver

from selenium.webdriver.common.by import By
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
from selenium.common.exceptions import TimeoutException
from selenium.common.exceptions import StaleElementReferenceException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


def remove_blank(data):
    return " ".join(data.split())


class Weibo(object):
    def __init__(self, scookie, proxy=None):
        """
        params:
        scookie[str]: cookie string
        proxy[tuple]: (proxy-type, prixy-addr),
                e.g.: ('socks5', '127.0.0.1:1080')
        """
        dcap = DesiredCapabilities.PHANTOMJS.copy()
        dcap["phantomjs.page.settings.userAgent"] = "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:41.0) Gecko/20100101 Firefox/41.0"
        dcap["phantomjs.page.settings.resourceTimeout"] = 1000
        dcap["phantomjs.page.settings.loadImages"] = False
        dcap["phantomjs.page.settings.disk-cache"] = True
        dcap["phantomjs.page.customHeaders.Cookie"] = scookie
        service_args = []
        if proxy:
            service_args.extend(['--proxy-type={type_}'.format(type_=proxy[0]),
                                '--proxy={addr}'.format(addr=proxy[1])])
        driver = webdriver.PhantomJS(service_args=service_args,
                                     desired_capabilities=dcap)
        driver.maximize_window()
        self.driver = driver
        self.cookies = self.format_cookie(scookie)

    def __del__(self):
        self.driver.close()

    def format_cookie(self, scookie):
        cookies = {}
        for item in scookie.strip().split(';'):
            key, value = item.split('=')
            cookies[key] = value
        return cookies

    def search(self, keyword, timeout=10, pages=10):
        datas = []
        page = 1
        while page < pages:
            url = 'http://s.weibo.com/weibo/{key}&b=1&page={p}'
            url = url.format(key=keyword, p=page)
            self.driver.get(url)
            self.driver.delete_all_cookies()
            for key, value in self.cookies.items():
                c = {}
                c['name'] = key
                c['value'] = value
                c['domain'] = '.weibo.com'
                c['path'] = '/'
                c['httponly'] = False
                c['secure'] = False
                self.driver.add_cookie(c)

            wait = WebDriverWait(self.driver, timeout)

            wait.until(EC.visibility_of_element_located(
                        (By.CSS_SELECTOR, "div.W_linecolor")))

            # load page, move to the top and then to the bottom 3 times in a row
            for _ in range(3):
                self.driver.execute_script("window.scrollTo(0, 0)")
                self.driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")
                time.sleep(0.1)

            divs = self.driver.find_elements(By.CSS_SELECTOR, "div.W_linecolor")
            for div in divs:
                nickname = div.find_element(By.XPATH, './/a[@nick-name]').text.strip()
                # date format like '2016-12-11 08:25'
                date = div.find_element(By.XPATH, './/a[@node-type="feed_list_item_date"]').text.strip()
                text = div.find_element(By.CSS_SELECTOR, 'p.comment_txt').text.strip()
                text = remove_blank(text)

                # print nickname, date, text
                datas.append([nickname, date, text])

            # judge exist page next
            try:
                wait.until(EC.visibility_of_element_located(
                            (By.CSS_SELECTOR, "a.page.next")))
            except TimeoutException:
                break
            else:
                page += 1

        return datas


if __name__ == "__main__":

    scookie = 'SINAGLOBAL=6633929784119.3545.1487296449103; _s_tentry=bobao.360.cn; Apache=1728386664005.3547.1489455618134; ULV=1489455618620:7:3:2:1728386664005.3547.1489455618134:1489372149493; login_sid_t=7ad7267bcf34e65bf0fe85460e40bd31; SWB=usrmdinst_8; WBtopGlobal_register_version=1998c4b1477c1700; SCF=Ar9X7idOHejWwVt7GvK9aIoqF5IRgx4RjJ56fMjfxbjyKVsX5wVsiQixxu1TOhX8DH4GiNGnJHF2nYB9ap_npZY.; SUB=_2A251w_7iDeTxGeNP4lYW9i3JzD2IHXVWuVcqrDV8PUNbmtBeLRLnkW-exa31susuSKHbN-wZi1-5gzEvRw..; SUBP=0033WrSXqPxfM725Ws9jqgMF55529P9D9W57dSMFsMpkTfr6whFAqffW5JpX5K2hUgL.Fo-p1KBNSoefS022dJLoI7UQ9sfDdJ-p; SUHB=0laV_lzCsDVZzT; ALF=1490078002; SSOLoginState=1489473202; un=hdmine1@163.com; UOR=bobao.360.cn,widget.weibo.com,www.findspace.name'
    weibo = Weibo(scookie, proxy=('http', '127.0.0.1:8080'))
    #keyword = 's2-045'
    keyword = 'CVE-2017-5638'
    datas = weibo.search(keyword)

    with open(keyword+'.txt', 'w') as f:
        for data in datas:
            line = '\t'.join(_ for _ in data) + '\n'
            f.write(line.encode('utf-8'))
```
