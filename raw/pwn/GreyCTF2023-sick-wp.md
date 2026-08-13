# GreyCTF 2023 sick

## 题目简述

目标是一个极小的无 libc x86-64 程序，`echo` 把最多 0x200 字节读入 0x20 字节栈缓冲区。可用 gadget 很少，但程序保留 `syscall; ret`，因此可通过 sigreturn-oriented programming 设置完整寄存器现场，先把代码段改为 RWX，再把 shellcode 写入其中执行。

## 解题过程

覆盖返回地址的偏移为 40。第一阶段让溢出返回到 `echo`，其后放置 `syscall; ret` 和伪造的 `rt_sigframe`。frame 设置：

```text
rax = 10          # mprotect
rdi = 0x400000
rsi = 0x4000
rdx = 7           # RWX
rip = syscall_ret
rsp = 0x4010d8    # 代码段中预置的 echo 返回跳板
```

第一阶段之后再次进入 `echo`。发送 14 个字符加换行，使 `read` 的返回值恰为 15；`rax=15` 在 x86-64 上对应 `rt_sigreturn`。紧随其后的 `syscall` 因而从栈上恢复伪造 frame，并执行 `mprotect(0x400000, 0x4000, 7)`。

恢复后的 `rsp` 位于已经可写的代码页，跳板再次调用 `echo`，其局部缓冲区落在约 `0x4010b0`。第三次输入写入 `execve("/bin/sh", 0, 0)` shellcode，填充到返回地址后把返回地址设为 `0x4010b0`。程序随即在代码页内执行 shellcode，读取 flag：

```text
grey{s1gr3tuRn_s4V3s_7He_D4Y_gsfs9761bk}
```

## 方法总结

SROP 适合只有 syscall、缺少寄存器设置 gadget 的小二进制。关键是利用 `read` 返回值把 `rax` 精确变成 15，并在 sigreturn frame 中一次性设置 mprotect 的所有参数。把 `rsp` 迁移到刚改成可写的代码页，还能让下一轮栈输入直接成为可执行载荷。
