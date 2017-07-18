---
title: selenium+phantomjs爬取twitter
date: 2017-03-13 20:42:27
categories: Python
tags:
    - twitter
    - selenium
    - phantomjs
    - 爬虫
---

## 简要介绍

前阵子Struts S2-045(CVE-2017-5638)漏洞可谓是赚足了眼球，风波现已慢慢平息，可以做一下漏洞数据方面的挖掘和分析，于是想到去twitter上抓一些数据看看，本以为是件很容易的事情，找到twitter搜索的url，写个脚本爬一下，解析数据保存，然后分析就可以了。终归只是自己这么想, 真正去做的时候才发现，twitter页面的加载是通过ajax请求去做的，简单的爬取，拿到的数据有限，难道又去分析ajax请求，想了下，太麻烦，也不通用。调用twitter api是一个可行的办法，但是考虑到注册以及api的限制，终归不是首选，于是就想起了`Selenium + Phantomjs`，这样就能解决js加载，抓取更多的数据了。

思路大致就是，`Selenium + Phantomjs` 模拟人的操作，不断滚动至页面最底端，加载新的内容，最后再将渲染的源码保存，后面再调用`BeautifulSoup`去解析，就可以实现抓取任意数量的内容了。

<!-- more -->

## 实现

* request.py

```python
#!/usr/bin/env python
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


class Wait_for_more_than_n_elements_to_be_present(object):
    def __init__(self, locator, count):
        self.locator = locator
        self.count = count

    def __call__(self, driver):
        try:
            elements = EC._find_elements(driver, self.locator)
            return len(elements) > self.count
        except StaleElementReferenceException:
            return False


def fetch_page(url, proxy=None, maxnum=1000):
    """selenium phantomjs fetch page
    params:
        url[str]: twitter url
        proxy[tuple]: (proxy-type, prixy-addr),
                e.g.: ('socks5', '127.0.0.1:1080')
    return:
        contents[str]:
    """
    timeout = 30

    dcap = DesiredCapabilities.PHANTOMJS.copy()
    dcap["phantomjs.page.settings.userAgent"] = "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:41.0) Gecko/20100101 Firefox/41.0"
    dcap["phantomjs.page.settings.resourceTimeout"] = 1000
    dcap["phantomjs.page.settings.loadImages"] = False
    dcap["phantomjs.page.settings.disk-cache"] = True

    service_args = ['--ignore-ssl-errors=true', '--ssl-protocol=TLSv1']
    if proxy:
        service_args.extend(['--proxy-type={type_}'.format(type_=proxy[0]),
                             '--proxy={addr}'.format(addr=proxy[1])])

    driver = webdriver.PhantomJS(service_args=service_args,
                                 desired_capabilities=dcap)
    driver.maximize_window()

    driver.get(url)

    # initial wait for the tweets to load
    wait = WebDriverWait(driver, timeout)
    wait.until(EC.visibility_of_element_located(
                (By.CSS_SELECTOR, "li[data-item-id]")))

    number_of_tweets = 0
    # scroll down to the last tweet until there is no more tweets loaded
    while number_of_tweets < maxnum:
        tweets = driver.find_elements(By.CSS_SELECTOR, "li[data-item-id]")
        number_of_tweets = len(tweets)
        print(number_of_tweets)

        # move to the top and then to the bottom 5 times in a row
        for _ in range(5):
            driver.execute_script("window.scrollTo(0, 0)")
            driver.execute_script("arguments[0].scrollIntoView(true);",
                                  tweets[-1])
            time.sleep(0.5)

        try:
            wait.until(Wait_for_more_than_n_elements_to_be_present(
                    (By.CSS_SELECTOR, "li[data-item-id]"), number_of_tweets))
        except TimeoutException:
            break

    contents = driver.page_source
    driver.close()

    return contents
```

* common.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals
import datetime
import time
import hashlib
import re
import sys


def now_time():
    return datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")


def timestamp_to_string(stamp):
    stamp = int(stamp)
    return time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(stamp))


def remove_blank(data):
    return " ".join(data.split())
```

* twitter.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals

from bs4 import BeautifulSoup
from common import remove_blank
from common import timestamp_to_string

from request import fetch_page


class Twitter(object):
    """Search twitter by keyword or user (@nickname)
    """
    def __init__(self, proxy=None):
        """
        params:
            proxy[tuple]: (proxy-type, prixy-addr),
                    e.g.: ('socks5', '127.0.0.1:1080')
        """
        self.keyword_url = 'https://twitter.com/search?q={keyword}&src=typd'
        self.user_url = 'https://twitter.com/{user}'
        self.proxy = proxy

    def parse_page(self, content):
        """
        parse urls: "https://twitter.com/search?q={keyword}&src=typd",
                    "https://twitter.com/{user}",
        return:
            value_list[list]: [(timestamp, url_full, content), ...]
        """
        parser = BeautifulSoup(content, 'lxml')
        value_list = []
        li_contents = parser.find_all('li', attrs={"data-item-type": "tweet"})
        for li_content in li_contents:
            try:
                content = li_content.find('p', class_="tweet-text").text
                # content = content.encode('utf-8', 'ignore')
                content = remove_blank(content)

                small = li_content.find('small', class_="time")
                url = small.a['href']
                url_full = "https://twitter.com" + url
                stamp = small.span['data-time']
                date = timestamp_to_string(stamp)

                value = (date, url_full, content)
                value_list.append(value)
            except Exception as ex:
                err_msg = "parse twitter content error: {}".format(ex)
                print err_msg
                continue

        return value_list

    def search_user(self, nickname, maxnum=1000):
        content = fetch_page(self.user_url.format(user=nickname),
                             proxy=self.proxy, maxnum=maxnum)
        return self.parse_page(content)

    def search_keyword(self, keyword, maxnum=1000):
        """
        """
        content = fetch_page(self.keyword_url.format(keyword=keyword),
                             proxy=self.proxy, maxnum=maxnum)
        with open(keyword+'.html', 'w') as f:
            f.write(content.encode('utf-8'))
        return self.parse_page(content)
```

* test.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals

from twitter import Twitter


def test_search_user():
    proxy = ('socks5', '127.0.0.1:1080')
    twitter = Twitter(proxy=proxy)
    user = 'sqlmap'
    with open('3.txt', 'w') as f:
        for d, u, v in twitter.search_user(user, maxnum=30):
            line = '{0}\t{1}\t{2}\n'.format(d, u, v)
            f.write(line.encode('utf-8'))


def test_search_keyword():
    proxy = ('socks5', '127.0.0.1:1080')
    twitter = Twitter(proxy=proxy)
    # keyword = 's2-045'
    keyword = 'struts'
    with open(keyword + '.txt', 'w') as f:
        for d, u, v in twitter.search_keyword(keyword, maxnum=200):
            line = '{0}\t{1}\t{2}\n'.format(d, u, v)
            f.write(line.encode('utf-8'))


if __name__ == "__main__":
    # test_search_user()
    test_search_keyword()
```
