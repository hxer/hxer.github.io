---
title: 【develop】nmap exit code 255
date: 2017-11-30 13:21:32
categories: develop
tags:
    - nmap
    - subprocess
---

使用python subprocess模块调用namp程序竟然返回255错误代码，如下所示:

```Python
In [4]: subprocess.check_output(['nmap'])
---------------------------------------------------------------------------
CalledProcessError                        Traceback (most recent call last)
<ipython-input-4-3ea255c65d35> in <module>()
----> 1 subprocess.check_output(['nmap'])

/usr/local/Cellar/python/2.7.14/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.pyc in check_output(*popenargs, **kwargs)
    217         if cmd is None:
    218             cmd = popenargs[0]
--> 219         raise CalledProcessError(retcode, cmd, output=output)
    220     return output
    221

CalledProcessError: Command '['nmap']' returned non-zero exit status 255
```

查看nmap在线文档[https://nmap.org/book/ncat-man-exit-code.html](https://nmap.org/book/ncat-man-exit-code.html)，返回码只有0, 1, 2三种，并没有提及255这个返回码，而且一般程序设计不会返回255这样的数字。于是只好查看nmap源码了。在nmap源码文件`nmap.cc`的`nmap_main`函数里面有大量返回码，包括: -1, 0, 1, 2。而不是只有文档中所描述的三种返回码。当nmap不带任何参数调用时，返回-1，也就是python显示的返回码255。-1和255有没有想到些什么，-1的补码就是255，难道程序的返回码做了类型转换，带着这个疑问，实验验证如下：
<!--more-->
### 实验验证

编写c程序如下:

```C
#include <stdio.h>

int main(){
    printf("return -1");
    return -1;
}
```

编译运行如下：

```
▶ ./test
return -1%
```

pyhon调用验证，显示返回状态为255，也就是-1的unsigned形式, 而不是直接显示-1的返回码。

```Python
In [2]: subprocess.check_output('./test')
---------------------------------------------------------------------------
CalledProcessError                        Traceback (most recent call last)
<ipython-input-2-20f1d6c2bb80> in <module>()
----> 1 subprocess.check_output('./test')

/usr/local/Cellar/python/2.7.14/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.pyc in check_output(*popenargs, **kwargs)
    217         if cmd is None:
    218             cmd = popenargs[0]
--> 219         raise CalledProcessError(retcode, cmd, output=output)
    220     return output
    221

CalledProcessError: Command './test' returned non-zero exit status 255
```

那么，python为什么不直接输出程序的返回码，而转成了正整数呢？参考：[https://docs.python.org/2/library/subprocess.html#subprocess.Popen.returncode](https://docs.python.org/2/library/subprocess.html#subprocess.Popen.returncode)

返回码为负数是有特殊含义的，-N表明子程序是被信号N所终止的。

