# gets

## 题目简述

程序只有一个 32 字节栈缓冲区和一次 `gets`：

```c
char buffer[32];
gets(buffer);
return sandbox();
```

二进制为 amd64、Full RELRO、无 canary、NX 开启、无 PIE。seccomp 默认终止进程，只允许 `mmap`、`open` 和 `read`；没有 `write`。程序也缺少常见的 CSU gadget，静态可用的关键片段只有 `pop rdi`、`pop rbp`、`leave; ret` 和一个 32 位加法 gadget。题面图片中的四行文字已经足以概括限制，因此不保留纯文字截图：

```text
BASIC RET2LIBC
NO LEAKS & FULL RELRO
LIBC 2.35 & NO GADGETS
GETS + POP RDI
```

## 解题过程

返回地址偏移为 40。第一步借 `gets` 把栈迁移到固定的 `.bss`：

```python
def gets(addr):
    return flat(
        pop_rdi,
        addr,
        exe.sym["gets"],
    )

def pivot(addr):
    return flat(
        pop_rbp,
        addr - 8,
        leave_ret,
    )

stage1 = b"A" * 40
stage1 += gets(0x404100)
stage1 += pivot(0x404100)
```

真正的难点是没有 libc 泄露，也没有足够的寄存器 gadget。官方方案利用 `gets` 内部 `_IO_getline_info` 的栈布局：把下一次 `gets` 的目标设为当前 `rsp` 上方的低地址，输入会覆盖 `_IO_getline_info` 自己保存的返回现场。该函数尾部依次恢复 `rbx`、`rbp`、`r12` 至 `r15` 后 `ret`，于是可以得到一批原二进制没有的寄存器控制。

调用 `gets` 的过程中，若干 libc 返回地址会自然落在 `.bss` 栈上。ASLR 会改变高位基址，但同一版本 libc 内部地址间的偏移固定。利用二进制中的：

```asm
add dword ptr [rbp - 0x3d], ebx
ret
```

可以给这些已存在的 libc 指针低 32 位加上已知差值，而无需知道其绝对地址。官方脚本依次合成了：

```text
pop rax ; pop rdx ; pop rbx ; ret
setcontext + 固定偏移
syscall ; ret
mmap 调用路径
```

有了 `setcontext`，在 `.bss` 中布置类 `ucontext` 结构即可一次恢复 `rdi`、`rsi`、`rdx`、`r8`、`r9`、`rsp` 和 `rip`。最终调用 `mmap` 创建 `0xdead000` 的 RWX 区域，再由 `gets` 写入并跳转到 shellcode。

seccomp 不允许输出，所以 shellcode 不能直接打印 flag。它只做三件事：

1. `open("flag.txt", 0)`；
2. `read(fd, flag_buffer, 0x40)`；
3. 接收一个索引和一个候选字符，比较 `flag[index]` 与候选值。

若不相等，代码继续等待下一个候选字符；若相等，则执行 `int3` 使连接异常关闭。远端连接是否关闭由此成为一位相等性 oracle。题目已说明 flag 字符集为大写字母、下划线与花括号，所以逐位置枚举有限字符集即可恢复：

```text
SEKAI{IT_KINDA_GETS_COMPLICATED}
```

官方脚本中的候选流存在少量网络缓冲偏移，需要在本地与远端分别校准；本质判据始终是“正确候选导致进程终止”。

## 方法总结

本题的核心不是传统 ret2libc，而是从库函数调用现场借寄存器，再用可控加法把随机化的现成 libc 指针改造成所需 gadget。Full RELRO 阻止 GOT 覆盖，缺少 `write` 又迫使利用链把程序崩溃转换成侧信道。遇到极少 gadget 时，库函数的保存寄存器、返回地址和错误行为本身都可能成为可利用资源。
