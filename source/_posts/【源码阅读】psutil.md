---
title: 【源码阅读】psutil
date: 2019-01-04 22:13:11
categories: source code
tags:
    - psutil
    - python
---


psutil(Python system and process utilities)是一个跨平台的进程管理和系统工具的python库，
可以获取进程、系统CPU，memory，disks，network等信息。psutil主要用于系统资源的监控，分析，以及对进程进行一定的管理。支持的系统有Linux, Windows, OSX, FreeBSD and Sun Solaris，32和64位系统都支持。

如果有写过python C扩展的经验，阅读psutil源码是比较轻松的，psutil源码文件结构清晰，根据操作系统平台的不同组织文件结构。假如没有这方面经验，也不影响，可以查阅官方文档的开发者指南，快速对psutil源码结构有个大致了解。

阅读psutil官方文档的[开发者指南](https://github.com/giampaolo/psutil/blob/master/DEVGUIDE.rst),开发者增加新的功能特性，通常涉及如下文件:

```
psutil/__init__.py                   # main psutil namespace
psutil/_ps{platform}.py              # python platform wrapper
psutil/_psutil_{platform}.c          # C platform extension
psutil/_psutil_{platform}.h          # C header file
psutil/tests/test_process|system.py  # main test suite
psutil/tests/test_{platform}.py      # platform specific test suite
```

在`psutil/__init__.py`文件中定义新的函数，然后在`psutil/_ps{platform}.py`实现函数功能，其中`{platform}`泛指不同的操作系统平台。涉及C扩展，在对应的c文件实现，例如`_pswindows.py`对应的c文件为`_psutil_windows.c`。

### 单点深入

以`pids()` 为切入点，深入分析底层实现方式。函数如下：

```python
def pids():
    """Return a list of current running PIDs."""
    return _psplatform.pids()
```

其中`_psplatform`表示当前的操作系统平台，例如windows平台下定义为`from . import _pswindows as _psplatform`，跟入`_pswindows.py`文件的`pids()`函数如下：

```python
from . import _psutil_windows as cext
pids = cext.pids
```

`pids()`函数功能由底层c扩展实现，继续跟入`_psutil_windows.c`文件

```c
/*
 * Return a Python list of all the PIDs running on the system.
 */
static PyObject *
psutil_pids(PyObject *self, PyObject *args) {
    DWORD *proclist = NULL;
    DWORD numberOfReturnedPIDs;
    DWORD i;
    PyObject *py_pid = NULL;
    PyObject *py_retlist = PyList_New(0);

    if (py_retlist == NULL)
        return NULL;
    proclist = psutil_get_pids(&numberOfReturnedPIDs);
    if (proclist == NULL)
        goto error;

    for (i = 0; i < numberOfReturnedPIDs; i++) {
        py_pid = Py_BuildValue("I", proclist[i]);
        if (!py_pid)
            goto error;
        if (PyList_Append(py_retlist, py_pid))
            goto error;
        Py_DECREF(py_pid);
    }

    // free C array allocated for PIDs
    free(proclist);
    return py_retlist;

error:
    Py_XDECREF(py_pid);
    Py_DECREF(py_retlist);
    if (proclist != NULL)
        free(proclist);
    return NULL;
}
```

函数实现核心为`psutil_get_pids`函数，定义在`arch/windows/process_info.c`文件中

```c
DWORD *
psutil_get_pids(DWORD *numberOfReturnedPIDs) {
    // Win32 SDK says the only way to know if our process array
    // wasn't large enough is to check the returned size and make
    // sure that it doesn't match the size of the array.
    // If it does we allocate a larger array and try again

    // Stores the actual array
    DWORD *procArray = NULL;
    DWORD procArrayByteSz;
    int procArraySz = 0;

    // Stores the byte size of the returned array from enumprocesses
    DWORD enumReturnSz = 0;

    do {
        procArraySz += 1024;
        if (procArray != NULL)
            free(procArray);
        procArrayByteSz = procArraySz * sizeof(DWORD);
        procArray = malloc(procArrayByteSz);
        if (procArray == NULL) {
            PyErr_NoMemory();
            return NULL;
        }
        if (! EnumProcesses(procArray, procArrayByteSz, &enumReturnSz)) {
            free(procArray);
            PyErr_SetFromWindowsErr(0);
            return NULL;
        }
    } while (enumReturnSz == procArraySz * sizeof(DWORD));

    // The number of elements is the returned size / size of each element
    *numberOfReturnedPIDs = enumReturnSz / sizeof(DWORD);

    return procArray;
}
```

调用[EnumProcesses api](https://docs.microsoft.com/en-us/windows/desktop/api/psapi/nf-psapi-enumprocesses)枚举进程，procArray为指向进程ID数组的指针，numberOfReturnedPIDs存储了进程数。