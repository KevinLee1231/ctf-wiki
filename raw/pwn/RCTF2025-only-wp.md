# only

## 题目简述

题目通过浮点/机器码输入进入可控执行路径。解法输入特定 double 值，送入一段 syscall stub，借助后续 read 把真正的 shellcode 放入可执行内存并读取 flag。

## 解题过程

### 关键观察

题目通过浮点/机器码输入进入可控执行路径。

### 求解步骤

#!/usr/bin/env python3
# -*- coding: utf-8 -*
import re
import os
from pwn import *
context(arch='amd64', os='linux', log_level='debug')
context.terminal = ['tmux', 'splitw', '-h']
local = 0
ip = "<challenge-host>"
port = 26100
ELF_PATH="chal"
if local:
    p = process(ELF_PATH)
else:
    p = remote(ip,port)
elf = ELF(ELF_PATH)
script = '''
    b *$rebase(0x1A40)
'''
def dbg():
    if local:
        gdb.attach(p,script)
    pause()
p.sendlineafter(b"3.exit", b'2')

## 方法总结

- 核心技巧：用特定浮点输入触发可控机器码路径。
- 识别信号：程序把浮点/字节解释结果用于执行或 syscall。
- 复用要点：第一阶段 shellcode 越短越好，目标是拿到第二阶段 read。
