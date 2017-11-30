---
title: 【Mac】解决ipython使用sqlite3报错问题
date: 2017-11-30 11:09:50
categories: solution
tags:
    - MAC
---

在pycharm上调试程序使用其自带python控制台，查看变量属性或做些简单的计算，出现报错如下：

```
self.ipython.history_manager.save_thread.pydev_do_not_trace = True #don't trace ipython history saving thread
```

python是brew安装的python 2.7.14版本，并安装了ipython==5.4.10, 乍一看是ipython的问题，第一感觉是ipython和pycharm不兼容，以为是ipython版本太高了导致不兼容，以前有出现过这样的问题。试了其他ipython版本，发现还是报错，那就不是ipython版本问题。

<!--more -->
于是命令行打开ipython, 提示警告如下：

```
/usr/local/lib/python2.7/site-packages/IPython/core/history.py:228: UserWarning: IPython History requires SQLite, your history will not be saved
  warn("IPython History requires SQLite, your history will not be saved")
```

也就是ipython不能使用sqlite来存储历史记录，应该是sqlite和 python的锅了，在linux上要先装sqlite3,再装python,以便python能使用sqlite接口。尝试`import sqlite3`，看看是否是这个问题，出现报错如下：

```
ImportError: dlopen(/usr/local/Cellar/python/2.7.14/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/_sqlite3.so, 2): Symbol not found: _sqlite3_enable_load_extension
  Referenced from: /usr/local/Cellar/python/2.7.14/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/_sqlite3.so
  Expected in: /usr/lib/libsqlite3.dylib
 in /usr/local/Cellar/python/2.7.14/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/_sqlite3.so
```

果然是导入错误，问题明确了，google一番：[https://stackoverflow.com/questions/41994449/symbol-not-found-sqlite3-enable-load-extension-sqlite-installed-via-homebrew](https://stackoverflow.com/questions/41994449/symbol-not-found-sqlite3-enable-load-extension-sqlite-installed-via-homebrew)

解决方案如下：

* `brew install sqlite` or `brew reinstall sqlite`

  ```
  his formula is keg-only, which means it was not symlinked into /usr/local,
  because macOS provides an older sqlite3.

  If you need to have this software first in your PATH run:
    echo 'export PATH="/usr/local/opt/sqlite/bin:$PATH"' >> ~/.zshrc

  For compilers to find this software you may need to set:
      LDFLAGS:  -L/usr/local/opt/sqlite/lib
      CPPFLAGS: -I/usr/local/opt/sqlite/include
  For pkg-config to find this software you may need to set:
      PKG_CONFIG_PATH: /usr/local/opt/sqlite/lib/pkgconfig
  ```

* vim .zshrc, 添加如下内容

  ```
  #sqlite
  export PATH="/usr/local/opt/sqlite/bin:$PATH"
  LDFLAGS="-L/usr/local/opt/sqlite/lib"
  CPPFLAGS="-I/usr/local/opt/sqlite/include"
  export PKG_CONFIG_PATH="/usr/local/opt/sqlite/lib/pkgconfig"
  ```

* source .zshrc

问题解决，ipython, pycharm均没问题。