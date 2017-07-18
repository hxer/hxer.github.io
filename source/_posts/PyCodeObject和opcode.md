---
title: PyCodeObject和opcode
date: 2017-04-01 18:59:20
categories: Python
tags:
    - PyCodeObject
    - opcode
---


## pyc文件

Python的原始代码在运行前都会被先编译成字节码，并把编译的结果保存到一个一个的PyCodeObject中，把PyCodeObject从内存中以marshal格式保存为文件，即pyc文件。python 内置库`dis`，可以把二进制反编译CPython bytecode。二进制对应的bytecode可以参考: [https://github.com/python/cpython/blob/2.7/Include/opcode.h](https://github.com/python/cpython/blob/2.7/Include/opcode.h) ,其中bytecode所代表的意义:[https://docs.python.org/2/library/dis.html](https://docs.python.org/2/library/dis.html)

<!-- more -->

## PyCodeObject

Python代码中Code.h中定义的PyCodeObject结构

```c
/* Bytecode object */  
typedef struct {  
    PyObject_HEAD  
    int co_argcount;        /* #arguments, except *args */  
    int co_nlocals;     /* #local variables */  
    int co_stacksize;       /* #entries needed for evaluation stack */  
    int co_flags;       /* CO_..., see below */  
    PyObject *co_code;      /* instruction opcodes */  
    PyObject *co_consts;    /* list (constants used) */  
    PyObject *co_names;     /* list of strings (names used) */  
    PyObject *co_varnames;  /* tuple of strings (local variable names) */  
    PyObject *co_freevars;  /* tuple of strings (free variable names) */  
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */  
    /* The rest doesn't count for hash/cmp */  
    PyObject *co_filename;  /* string (where it was loaded from) */  
    PyObject *co_name;      /* string (name, for reference) */  
    int co_firstlineno;     /* first source line number */  
    PyObject *co_lnotab;    /* string (encoding addr<->lineno mapping) See
                   Objects/lnotab_notes.txt for details. */  
    void *co_zombieframe;     /* for optimization only (see frameobject.c) */  
    PyObject *co_weakreflist;   /* to support weakrefs to code objects */  
} PyCodeObject;
```

各字段的意义:

* argcount：参数的个数
* nlocals：局部变量的个数（包含参数在内）
* stacksize：堆栈的大小
* flags：用来表示参数中是否有`*args`或者 `**kwargs`
* code：字节码
* names：全局变量，函数，类，类的方法的名称
* varnames：局部变量的名称（包含参数）
* consts：一个常量表，在marshal.c中有定义所有的类型

```c
#define TYPE_NULL               '0'  
#define TYPE_NONE               'N'  
#define TYPE_FALSE              'F'  
#define TYPE_TRUE               'T'  
#define TYPE_STOPITER           'S'  
#define TYPE_ELLIPSIS           '.'  
#define TYPE_INT                'i'  
#define TYPE_INT64              'I'  
#define TYPE_FLOAT              'f'  
#define TYPE_BINARY_FLOAT       'g'  
#define TYPE_COMPLEX            'x'  
#define TYPE_BINARY_COMPLEX     'y'  
#define TYPE_LONG               'l'  
#define TYPE_STRING             's'  
#define TYPE_INTERNED           't'  
#define TYPE_STRINGREF          'R'  
#define TYPE_TUPLE              '('  
#define TYPE_LIST               '['  
#define TYPE_DICT               '{'  
#define TYPE_CODE               'c'  
#define TYPE_UNICODE            'u'  
#define TYPE_UNKNOWN            '?'  
#define TYPE_SET                '<'  
#define TYPE_FROZENSET          '>'
```

所有的PyCodeObject都是通过调用以下的函数得以运行的：`PyObject * PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)`, 这个函数是Python的一个重量极的函数，它的作用即是执行中间码，Python的代码都是通过调用这个函数来运行的

PyFrameObject这个数据结构：

```c
typedef struct _frame {  
    PyObject_VAR_HEAD  
    struct _frame *f_back;  /* previous frame, or NULL */  
    PyCodeObject *f_code;   /* code segment */  
    PyObject *f_builtins;   /* builtin symbol table (PyDictObject) */  
    PyObject *f_globals;    /* global symbol table (PyDictObject) */  
    PyObject *f_locals;     /* local symbol table (any mapping) */  
    PyObject **f_valuestack;    /* points after the last local */  
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */  
    PyObject **f_stacktop;  
    PyObject *f_trace;      /* Trace function */  


    /* If an exception is raised in this frame, the next three are used to
     * record the exception info (if any) originally in the thread state.  See
     * comments before set_exc_info() -- it's not obvious.
     * Invariant:  if _type is NULL, then so are _value and _traceback.
     * Desired invariant:  all three are NULL, or all three are non-NULL.  That
     * one isn't currently true, but "should be".
     */  
    PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;  


    PyThreadState *f_tstate;  
    int f_lasti;        /* Last instruction if called */  
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */  
    int f_lineno;       /* Current line number */  
    int f_iblock;       /* index in f_blockstack */  
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */  
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */  
} PyFrameObject;
```

重点关注下以下的成员：

```c
PyObject **f_stacktop;  

PyCodeObject *f_code;   /* code segment */  
PyObject *f_globals;    /* global symbol table (PyDictObject) */  
PyObject *f_locals;     /* local symbol table (any mapping) */
```

可见PyFrameObject中包含一个PyCodeObject的结构体指针f_code,PyFrameObject中的f_globals和f_locals这两个dict对象分别用于保存全局对象和局部对象，可以说，PyFrameObject结构就是PyCodeObject的运行环境，PyCodeObject结构是静态的，创建以后就一般不会再变，而PyFrameObject则是动态的，在创建以后,f_globals和f_locals这两个dict都经常会发生变化，并且一个PyCodeObject可能对应好几个的PyFrameObject中。

## 实例演示

* 新建test.py

```python
import dis  
myglobal = True  

def add(a):  
    b = 1  
    a += b  
    return a  

class world:  
    def __init__(self):  
        pass  
    def sayHello(self):  
        print 'hello,world'  

w = world()  
w.sayHello()  
```
* 编译成pyc

```
python -m compileall test.py
```

* 分析test.pyc

showfile.py

```python
import dis, marshal, struct, sys, time, types  

def show_file(fname):  
    f = open(fname, "rb")  
    magic = f.read(4)  
    moddate = f.read(4)  
    modtime = time.asctime(time.localtime(struct.unpack('L', moddate)[0]))  
    print "magic %s" % (magic.encode('hex'))  
    print "moddate %s (%s)" % (moddate.encode('hex'), modtime)  
    code = marshal.load(f)  
    show_code(code)  

def show_code(code, indent=''):  
    old_indent = indent  
    print "%s<code>" % indent  
    indent += '   '  
    print "%s<argcount> %d </argcount>" % (indent, code.co_argcount)  
    print "%s<nlocals> %d</nlocals>" % (indent, code.co_nlocals)  
    print "%s<stacksize> %d</stacksize>" % (indent, code.co_stacksize)  
    print "%s<flags> %04x</flags>" % (indent, code.co_flags)  
    show_hex("code", code.co_code, indent=indent)  
    print "%s<dis>" % indent  
    dis.disassemble(code)  
    print "%s</dis>" % indent  

    print "%s<names> %r</names>" % (indent, code.co_names)  
    print "%s<varnames> %r</varnames>" % (indent, code.co_varnames)  
    print "%s<freevars> %r</freevars>" % (indent, code.co_freevars)  
    print "%s<cellvars> %r</cellvars>" % (indent, code.co_cellvars)  
    print "%s<filename> %r</filename>" % (indent, code.co_filename)  
    print "%s<name> %r</name>" % (indent, code.co_name)  
    print "%s<firstlineno> %d</firstlineno>" % (indent, code.co_firstlineno)  

    print "%s<consts>" % indent  
    for const in code.co_consts:  
        if type(const) == types.CodeType:  
            show_code(const, indent+'   ')  
        else:  
            print "   %s%r" % (indent, const)  
    print "%s</consts>" % indent  

    show_hex("lnotab", code.co_lnotab, indent=indent)  
    print "%s</code>" % old_indent  

def show_hex(label, h, indent):  
    h = h.encode('hex')  
    if len(h) < 60:  
        print "%s<%s> %s</%s>" % (indent, label, h,label)  
    else:  
        print "%s<%s>" % (indent, label)  
        for i in range(0, len(h), 60):  
            print "%s   %s" % (indent, h[i:i+60])  
        print "%s</%s>" % (indent, label)  

show_file(sys.argv[1])
```

```
showfile.py test.pyc > test.xml  
```

* test.xml

```xml
magic 03f30d0a
moddate 9c4cdf58 (Sat Apr  1 14:45:48 2017)
<code>
   <argcount> 0 </argcount>
   <nlocals> 0</nlocals>
   <stacksize> 3</stacksize>
   <flags> 0040</flags>
   <code>
      6400006401006c00005a00006501005a02006402008400005a0300640300
      640500640400840000830000595a04006504008300005a05006505006a06
      008300000164010053
   </code>
   <dis>
  1           0 LOAD_CONST               0 (-1)
              3 LOAD_CONST               1 (None)
              6 IMPORT_NAME              0 (dis)
              9 STORE_NAME               0 (dis)

  2          12 LOAD_NAME                1 (True)
             15 STORE_NAME               2 (myglobal)

  4          18 LOAD_CONST               2 (<code object add at 0x10e0b7ab0, file "test.py", line 4>)
             21 MAKE_FUNCTION            0
             24 STORE_NAME               3 (add)

  9          27 LOAD_CONST               3 ('world')
             30 LOAD_CONST               5 (())
             33 LOAD_CONST               4 (<code object world at 0x10e0b7db0, file "test.py", line 9>)
             36 MAKE_FUNCTION            0
             39 CALL_FUNCTION            0
             42 BUILD_CLASS         
             43 STORE_NAME               4 (world)

 15          46 LOAD_NAME                4 (world)
             49 CALL_FUNCTION            0
             52 STORE_NAME               5 (w)

 16          55 LOAD_NAME                5 (w)
             58 LOAD_ATTR                6 (sayHello)
             61 CALL_FUNCTION            0
             64 POP_TOP             
             65 LOAD_CONST               1 (None)
             68 RETURN_VALUE        
   </dis>
   <names> ('dis', 'True', 'myglobal', 'add', 'world', 'w', 'sayHello')</names>
   <varnames> ()</varnames>
   <freevars> ()</freevars>
   <cellvars> ()</cellvars>
   <filename> 'test.py'</filename>
   <name> '<module>'</name>
   <firstlineno> 1</firstlineno>
   <consts>
      -1
      None
      <code>
         <argcount> 1 </argcount>
         <nlocals> 2</nlocals>
         <stacksize> 2</stacksize>
         <flags> 0043</flags>
         <code> 6401007d01007c00007c0100377d00007c000053</code>
         <dis>
  5           0 LOAD_CONST               1 (1)
              3 STORE_FAST               1 (b)

  6           6 LOAD_FAST                0 (a)
              9 LOAD_FAST                1 (b)
             12 INPLACE_ADD         
             13 STORE_FAST               0 (a)

  7          16 LOAD_FAST                0 (a)
             19 RETURN_VALUE        
         </dis>
         <names> ()</names>
         <varnames> ('a', 'b')</varnames>
         <freevars> ()</freevars>
         <cellvars> ()</cellvars>
         <filename> 'test.py'</filename>
         <name> 'add'</name>
         <firstlineno> 4</firstlineno>
         <consts>
            None
            1
         </consts>
         <lnotab> 000106010a01</lnotab>
      </code>
      'world'
      <code>
         <argcount> 0 </argcount>
         <nlocals> 0</nlocals>
         <stacksize> 1</stacksize>
         <flags> 0042</flags>
         <code> 6500005a01006400008400005a02006401008400005a03005253</code>
         <dis>
  9           0 LOAD_NAME                0 (__name__)
              3 STORE_NAME               1 (__module__)

 10           6 LOAD_CONST               0 (<code object __init__ at 0x10e0b7c30, file "test.py", line 10>)
              9 MAKE_FUNCTION            0
             12 STORE_NAME               2 (__init__)

 12          15 LOAD_CONST               1 (<code object sayHello at 0x10e0b7d30, file "test.py", line 12>)
             18 MAKE_FUNCTION            0
             21 STORE_NAME               3 (sayHello)
             24 LOAD_LOCALS         
             25 RETURN_VALUE        
         </dis>
         <names> ('__name__', '__module__', '__init__', 'sayHello')</names>
         <varnames> ()</varnames>
         <freevars> ()</freevars>
         <cellvars> ()</cellvars>
         <filename> 'test.py'</filename>
         <name> 'world'</name>
         <firstlineno> 9</firstlineno>
         <consts>
            <code>
               <argcount> 1 </argcount>
               <nlocals> 1</nlocals>
               <stacksize> 1</stacksize>
               <flags> 0043</flags>
               <code> 64000053</code>
               <dis>
 11           0 LOAD_CONST               0 (None)
              3 RETURN_VALUE        
               </dis>
               <names> ()</names>
               <varnames> ('self',)</varnames>
               <freevars> ()</freevars>
               <cellvars> ()</cellvars>
               <filename> 'test.py'</filename>
               <name> '__init__'</name>
               <firstlineno> 10</firstlineno>
               <consts>
                  None
               </consts>
               <lnotab> 0001</lnotab>
            </code>
            <code>
               <argcount> 1 </argcount>
               <nlocals> 1</nlocals>
               <stacksize> 1</stacksize>
               <flags> 0043</flags>
               <code> 640100474864000053</code>
               <dis>
 13           0 LOAD_CONST               1 ('hello,world')
              3 PRINT_ITEM          
              4 PRINT_NEWLINE       
              5 LOAD_CONST               0 (None)
              8 RETURN_VALUE        
               </dis>
               <names> ()</names>
               <varnames> ('self',)</varnames>
               <freevars> ()</freevars>
               <cellvars> ()</cellvars>
               <filename> 'test.py'</filename>
               <name> 'sayHello'</name>
               <firstlineno> 12</firstlineno>
               <consts>
                  None
                  'hello,world'
               </consts>
               <lnotab> 0001</lnotab>
            </code>
         </consts>
         <lnotab> 06010902</lnotab>
      </code>
      ()
   </consts>
   <lnotab> 0c010602090513060901</lnotab>
</code>
```

可以看到，整个test.pyc就是一个嵌套的PyCodeObject结构的组合，对于每个函数，或者类的方法，都会生成一个对应的PyCodeObject结构，并且模块还会生成额外的一个PyCodeObject结构,pyc文件中一共保存了5个PyCodeObject结构：

* 最外层的为模块test的PyCodeObject，其中的代码在import或者直接运行时会得到执行。
* 在module的PyCodeObject中嵌套了add函数的PyCodeObject结构以及world类的PyCodeObject结构。
* 在world类的PyCodeObject结构中，又嵌套了__init__方法和sayHello方法的PyCodeObject结构。

### 解析部分产生的opcode

```
1           0 LOAD_CONST               0 (-1) #加载consts数组中索引为0处的值，这里为数值-1  
             3 LOAD_CONST               1 (None) #加载consts数组中索引为1处的值，这里为None  


             6 IMPORT_NAME              0 (dis) #加载dis模块：names[0]即为"dis"  
             9 STORE_NAME               0 (dis) #将模块保存到一个dict中，这个dict专门用来保存局部变量的值，key为names[0],即"dis"  


 2          12 LOAD_NAME                1 (True) #将names[1]，即True压栈。  
            15 STORE_NAME               2 (myglobal) #将栈顶的元素，即True保存到locals['myglobal']中,names[2]即为字符串'myglobal'  


 4          18 LOAD_CONST               2 (<code object add at 024E3B60, file "test.py", line 4>) #将consts[2]，即add函数的PyCodeObject压栈。  
            21 MAKE_FUNCTION            0 #通过add函数的PyCodeObject创建一个函数，并压入栈顶。  
            24 STORE_NAME               3 (add) #将创建的函数出栈并保存到locals['add']中  


 9          27 LOAD_CONST               3 ('world') #consts[3],即"world"入栈  
            30 LOAD_CONST               5 (()) #consts[5],即空数组入栈  
            33 LOAD_CONST               4 (<code object world at 024E3650, file "test.py", line 9>) #将consts[4]，即world的PyCodeObject入栈。  
            36 MAKE_FUNCTION            0 #创建函数  
            39 CALL_FUNCTION            0 #调用刚创建的函数，用于初始化类，会返回一个dict到栈顶，这个dict中保存了类的方法以及一些全局变量，  
                                        #具体的实现要看world类的PyCodeObject中的opcode  
            42 BUILD_CLASS               #创建类，注意BUILD_CLASS会用到栈中的三个对象，这里分别为：保存在dict中的类的信息，基类数组，这里为空数组(),类的名称："world"  
            43 STORE_NAME               4 (world) #将类保存到locals['world']中  


15          46 LOAD_NAME                4 (world)  
            49 CALL_FUNCTION            0  
            52 STORE_NAME               5 (w)  
#以上三行代码创建一个world对象  
16          55 LOAD_NAME                5 (w)  
            58 LOAD_ATTR                6 (sayHello)  
            61 CALL_FUNCTION            0  
#以上三行代码调用w.sayHello()  
            64 POP_TOP               
            65 LOAD_CONST               1 (None)  
            68 RETURN_VALUE          
```
