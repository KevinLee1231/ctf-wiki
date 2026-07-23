# UMDCTF2022 Tracestory Writeup

## 题目简述

程序把用户输入复制到一页可读、可写、可执行的匿名内存后直接调用，看似是普通 shellcode 题。但执行前加载的 seccomp 白名单不允许 `read`、`write`、`open` 或 `execve`，只保留了 `ptrace`、`gettimeofday` 等少量系统调用。

在安装 seccomp 之前，程序已经 `fork` 出一个子进程。子进程不受父进程后续过滤器影响，并循环读取 `readstory.txt`。解法是让父进程中的受限 shellcode 使用 `ptrace` 热补丁这个未受限子进程，使它改读并输出 `flag`。

## 解题过程

二进制为 amd64、Partial RELRO、启用 Canary 与 NX、关闭 PIE且未去符号。主函数的流程是：

```text
fork
├─ 子进程：read_story()，循环 open/read/close
└─ 父进程：读取 shellcode → setup_seccomp() → 调用 shellcode
```

`setup_seccomp` 默认拒绝系统调用，但显式允许编号 101 的 `ptrace` 和编号 96 的 `gettimeofday`。子进程是在过滤器加载前创建的，所以父进程仍可通过 `ptrace` 修改它的代码和只读数据。

远端子进程只有 PID 为偶数时才执行读文件分支；若服务端打印出的 child PID 为奇数，就重新连接。得到偶数 PID 后，shellcode 完成以下动作：

1. `PTRACE_ATTACH` 附加子进程。
2. 因 `wait` 不在白名单中，反复调用 `gettimeofday` 做一个短暂等待，让附加产生的停止信号生效。
3. 从 `0x401802` 起写入 32 字节补丁：前部填充 NOP，末尾执行 `mov rdi, rsi`，随后保留原有的 `call puts@plt`（地址 `0x401822`）。这样 `read` 返回后，读取缓冲区会直接交给 `puts`。
4. 把 `0x402011` 处的 `readstory.txt` 改成 `flag\0`。
5. 把 `0x401827` 开始的 10 字节退出路径改成 NOP。可分别在 `0x401827` 与 `0x401829` 写入 8 字节 NOP，使两次写入重叠并完整覆盖 `mov edi, 1; call exit`，随后 `PTRACE_DETACH`。
6. 父进程原地循环，给子进程继续运行和输出的时间。

补丁生成和 `ptrace` 写入的核心可以概括为：

```python
patch = asm("mov rdi, rsi")
patch = asm("nop") * (0x401822 - 0x401802 - len(patch)) + patch
words = [u64(patch[i:i + 8]) for i in range(0, 32, 8)]

# shellcode 中依次调用：
# ptrace(PTRACE_ATTACH, child_pid, 0, 0)
# ptrace(PTRACE_POKETEXT, child_pid, 0x401802 + 8*i, words[i])
# ptrace(PTRACE_POKETEXT, child_pid, 0x402011, 0x67616c66)
# ptrace(PTRACE_POKETEXT, child_pid, 0x401827, 0x9090909090909090)
# ptrace(PTRACE_POKETEXT, child_pid, 0x401829, 0x9090909090909090)
# ptrace(PTRACE_DETACH, child_pid, 0, 0)
```

完整公开利用脚本还处理了远端偶数 PID 重试和本地调试差异，可在 [datajerk 的 Tracestory exploit](https://github.com/datajerk/ctf-write-ups/blob/master/umdctf2022/trace_story/exploit.py) 中查看；上述步骤已经包含取得 flag 所需的全部机制和补丁位置。

子进程下一次循环会打开 `flag`，读取后调用 `puts`：

```text
UMDCTF{Tr4C3_Thr0Ugh_3v3rYth1NG}
```

## 方法总结

seccomp 只约束安装过滤器的进程及其后续派生关系，并不会追溯约束此前已经 fork 出去的兄弟执行流。本题的关键不是在白名单内直接完成文件 I/O，而是把允许的 `ptrace` 变成跨进程代码写原语，再借未过滤子进程代为执行 `open`、`read` 和 `write`。
