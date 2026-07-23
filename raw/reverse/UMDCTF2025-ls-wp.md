# UMDCTF 2025 - ls

## 题目简述

程序 `fork` 出父子两个进程。子进程看似不断调用与参数完全不匹配的 Linux 系统调用；
父进程则通过 `ptrace(PTRACE_SYSCALL)` 截获每次 syscall entry，并用输入 flag 的下一
字节异或 `orig_rax`：

```c
regs.orig_rax = regs.orig_rax ^ buf[i];
i++;
ptrace(PTRACE_SETREGS, pid, 0, &regs);
```

子进程实际上提前准备了另一个系统调用所需的寄存器。只有输入字符正确时，错误的
syscall number 才会被异或成目标 number，让程序继续执行。

## 解题过程

每一位都满足简单关系：

```text
flag_byte = apparent_syscall_number XOR intended_syscall_number
```

开头几次调用都想执行 `write`，其 x86-64 syscall number 为 1。例如：

```c
rmdir((char *)1);               // rmdir = 84
truncate((char *)1, (long)buf); // truncate = 76
msgsnd(1, buf, ..., 0);         // msgsnd = 69
syscall(66, 1, "lister!\n", 8);
```

分别计算：

```text
84 XOR 1 = 85 = 'U'
76 XOR 1 = 77 = 'M'
69 XOR 1 = 68 = 'D'
66 XOR 1 = 67 = 'C'
```

继续按子进程预先设置的参数判断真实意图，即可逐字节恢复。中间几组还用实际执行
结果验证了解码是否正确：

1. `setsid` 被改成 `read`，随后若干调用被改成 `open`、`write`、`dup2` 和
   `getdents`，完成一个简化的目录列举流程，对应 `ptrace-is-`；
2. `access` 被改成 `gettimeofday`，结果必须落在 2025 年范围内；`getpgrp` 被改成
   `write`，输出 `Test 1 success.`；
3. 显式 syscall 334 经字符 `k` 变为 293，即 `pipe2`；`getresgid` 变成
   `write`，`linkat` 变成 `dup3`，`setuid` 变成 `read`，再确认管道中读回
   `not_the_flag`；
4. syscall 34 经字符 `+` 变成 9，即 `mmap`，分配 RWX 页。程序写入
   `0xc3`（`ret`）并调用它，完成第三项检查。

把每次“表面 syscall”与其参数体现的“真实 syscall”逐项异或，得到：

```text
UMDCTF{ptrace-is-funky-i5n+-1t}
```

需要注意，附件把第一项时间检查硬编码为
`1735707600 <= tv_sec <= 1767243600`，即 2025 年附近。赛后在其他年份直接运行时，
即使 flag 正确也可能中途退出；这不影响静态恢复结果。

## 方法总结

本题利用 `ptrace` 在 syscall entry 阶段修改 `orig_rax`，把输入字符当成系统调用号的
一次性 XOR 密钥。逆向时不能只相信源码里的函数名，应结合寄存器参数、返回值用途和
后续检查判断真正想执行的调用。恢复公式一旦确定，剩余工作就是逐事件建立
“表面编号 → 目标编号 → 字符”的映射；时间限制属于运行时环境约束，不应被误判为
flag 的一部分。
