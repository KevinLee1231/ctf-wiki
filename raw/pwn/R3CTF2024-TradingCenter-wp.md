# TradingCenter

## 题目简述

程序把交易小游戏、文件管理器和受 seccomp 限制的 shellcode 执行组合在一起，并由父子进程分担功能。直接在子进程中执行 `execve` 会被沙箱阻止，但父进程仍可调用 `ptrace`。目标是先获得足够资金和地址泄露，在父进程中构造 ROP，通过 `ptrace` 修改子进程的沙箱安装函数，随后让子进程执行未受限 shellcode。

仓库只保留题面，没有完整附件源码；下面的链条依据公开完整利用脚本整理，所有地址偏移都必须在题目原二进制和配套 libc 上重新确认。

## 解题过程

### 利用固定答案积累资金

小游戏要求在 0 到 3 中选择一个值。服务实例内的正确值不会在每轮重新生成，因此先依次尝试四个候选，找到没有返回 `NONONO!!!` 的值，再重复提交它。约 255 轮内即可把余额提升到开启后续功能所需的 `134217728`。

```python
winner = next(v for v in range(4) if game(v).is_win)
while money < 134217728:
    money = game(winner).money
```

解锁文件管理器后取得子进程 PID。公开本地复现把原有文件输出路径简化成直接输出 `getpid()`；远端利用的目标仍是可靠获得稍后要附加的子进程 PID。

### 泄露父进程地址并装入 ROP

另一个功能允许提交一小段 shellcode，但初始执行环境和寄存器受限。公开利用从 `fs` 指向的线程本地存储取出栈相关地址，转用 `writev` 泄露一段内存，再用 `readv` 把长 ROP 链读入可控区域。

泄露内容包括：

- FS/TLS 基址与可用栈位置；
- 返回地址或程序指针；
- libc 指针。

据此计算配套 libc 基址，并准备 `ptrace`、`wait4`、syscall gadget 与可写缓冲区。这里的关键不是固定 gadget 偏移，而是先把短 shellcode原语扩展为可持续的 ROP 输入与输出通道。

### 从父进程修改子进程代码

ROP 链按以下顺序操作已知 PID：

```text
ptrace(PTRACE_ATTACH, pid, 0, 0)
wait4(pid, ...)
ptrace(PTRACE_GETREGS, pid, 0, regs)
```

从 `user_regs_struct` 中的指令指针恢复子进程 PIE 基址。公开附件中，沙箱安装函数入口位于：

```text
child_pie + 0x14f6
```

入口第一条是 `push rbp`。使用：

```text
ptrace(PTRACE_POKEDATA, pid, sandbox_entry, 0x...c3)
```

把开头改为 `ret`，再执行：

```text
ptrace(PTRACE_CONT, pid, 0, 0)
```

这样不是绕过已经安装的 seccomp，而是让后续调用“安装沙箱”的函数立即返回，沙箱根本不会进入生效状态。`0x14f6` 只属于公开题目构建，应通过同版本反汇编和寄存器泄露重新定位。

### 在子进程执行最终 shellcode

父进程完成补丁并恢复子进程后，再次进入 shellcode 功能。程序仍会调用沙箱安装函数，但其入口已经是 `ret`。此时提交普通的：

```asm
push 0x68732f6e69622f
mov rdi, rsp
xor esi, esi
xor edx, edx
mov eax, 59
syscall
```

即可在子进程中执行 `/bin/sh`。取得交互后读取 flag。

完整的短 shellcode、泄露布局和 ptrace ROP 链见 [R3CTF Pwn 复现记录](https://xshhc.github.io/2024/07/29/2024_R3CTF_part1/)。本文已把外链中的关键逻辑归纳为“固定小游戏状态、短 shellcode 扩展、父进程 ptrace、子进程禁用 seccomp、二次 shellcode”五个阶段。

## 方法总结

seccomp 只约束安装它的进程和之后发生的系统调用，不会自动阻止另一个有权限的进程用 `ptrace` 改写其代码。本题利用的是进程间信任边界：父进程保留了调试能力，子进程又会重复调用可修改的沙箱初始化函数。分析这类题时应画清父子进程、PID、谁被限制以及谁能调试谁；仅枚举子进程允许的 syscall 不足以发现这条路径。
