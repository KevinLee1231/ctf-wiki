# 只读文件系统

## 题目简述

题目已经允许通过漏洞执行自定义机器码，但容器文件系统只读。判题程序并不检查输出，而是监视进程的 `/proc/<pid>/exe`，要求它最终指向题目给定的 `hello` ELF。常规的“把 ELF 写到磁盘再 `execve`”不可行，突破口是 Linux `memfd_create`：在内存中创建匿名文件，写入 ELF 后直接执行。

## 解题过程

### 构造内存 ELF

完整流程可用下面的 C 语义表示：

```c
int fd = memfd_create("hello", 0);

while (received < elf_size) {
    ssize_t n = read(sock, buffer, sizeof(buffer));
    if (n <= 0) _exit(1);
    write_all(fd, buffer, n);
    received += n;
}

lseek(fd, 0, SEEK_SET);
char *argv[] = {"hello", NULL};
char *envp[] = {NULL};
execveat(fd, "", argv, envp, AT_EMPTY_PATH);
```

`memfd_create` 返回的文件描述符对应匿名、可寻址的内存文件，不需要在只读根文件系统中创建目录项。`execveat` 配合空路径和 `AT_EMPTY_PATH` 可以直接把该 fd 当作可执行文件。

若目标环境不便调用 `execveat`，也可尝试执行 `/proc/self/fd/<fd>`；该路径只是已打开描述符的视图，同样没有向根文件系统写入新文件。

### 可靠接收 payload

利用代码不能假设一次 `read` 就收到完整 ELF。TCP 是字节流，单次读取可能只返回任意前缀；shellcode 必须循环读取，或者让发送端先传长度，再严格累计到目标长度。写入 memfd 时也要处理短写：

```c
while (off < n) {
    ssize_t written = write(fd, buffer + off, n - off);
    if (written <= 0) _exit(1);
    off += written;
}
```

将题目提供的 `hello` ELF 全部写入后执行，进程映像被替换，`/proc/<pid>/exe` 随之指向 memfd 中的新 ELF。监控器因此通过校验并返回 flag。

## 方法总结

- 核心技巧：用 `memfd_create`、循环写入和 `execveat(..., AT_EMPTY_PATH)` 在只读文件系统上完成进程映像替换。
- 识别信号：已有代码执行，但限制写盘；目标又要求运行一个完整 ELF，而非仅打印其输出。
- 复用要点：只读文件系统不等于不能创建匿名内存对象；网络收发和 `write` 都可能短读短写，利用代码必须显式循环。
