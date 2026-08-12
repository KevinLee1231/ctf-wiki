# 传达不到的文件

## 题目简述

题目提供一个受限 Linux 虚拟环境，共两问。`/chall` 是只能执行、不能直接读取的特权程序，第一段 flag 硬编码在其中；输入正确后，程序再执行选手提供的 shellcode，但通过 seccomp user notification 限制文件打开。第二段 flag 位于无权直接读取的 `/flag2`。

## 解题过程

### 读不到：用 ptrace 把 read 改成 write

文件没有读权限并不妨碍内核把它映射为正在运行的进程。创建子进程，在 `execve("/chall")` 前调用 `PTRACE_TRACEME`，父进程便可以跟踪子进程：

```c
pid_t child = fork();
if (child == 0) {
    ptrace(PTRACE_TRACEME, 0, NULL, NULL);
    execl("/chall", "/chall", NULL);
    _exit(1);
}
```

父进程先用 `PTRACE_SYSCALL` 跟踪动态加载阶段。观察用于设置代码段权限的 `mprotect`，即可推算 PIE 映射基址。随后对题目自己的入口代码使用 `PTRACE_SINGLESTEP`，在一次已有的 `write` 调用前保存寄存器，在即将执行输入 `read` 的系统调用前改写为：

```text
rax = SYS_write
rdi = 1
rsi = text_base
rdx = 0x6000
```

最后用 `PTRACE_SETREGS` 写回寄存器并让系统调用继续。原本从标准输入读数据的指令被变成向标准输出写出整个映射区，因而能保存一份 `/chall` 的内存镜像。

在镜像中搜索 `flag{`，或直接反汇编入口处的比较逻辑，得到第一段：

```text
flag{ptr4ce_m3_4nd_1_w1ll_4lways_b3_th3r3_f0r_u}
```

这一方法没有改变文件权限，而是利用调试接口修改正在运行的进程状态。

### 分析 seccomp user notification

输入第一段 flag 后，`/chall` fork 出 shellcode 子进程。子进程安装带 listener 的 seccomp 过滤器，把通知 FD 通过 Unix socket 传给父进程，再降权到 UID 1337 执行 shellcode。

大多数文件打开相关系统调用都被过滤器禁止。传统 `open` 被转交给高权限父进程审查，只有满足以下条件的路径才放行：

```c
strncmp(path, "/proc", 5) == 0 &&
strstr(path, ".") == NULL &&
strstr(path, "self") == NULL
```

父进程会替子进程打开通过检查的路径，再用 `SECCOMP_IOCTL_NOTIF_ADDFD` 把 FD 注入子进程。直接传 `/flag2` 会失败，`/proc/self/fd/...` 也被显式阻止。

### 打不开：用 open_tree 制造可放行的 proc 路径

过滤器遗漏了 `open_tree`。不带 `OPEN_TREE_CLONE` 调用它时，可以取得指向路径的 `O_PATH` 文件描述符：

```c
int pathfd = syscall(SYS_open_tree, AT_FDCWD, "/flag2", 0);
```

`O_PATH` FD 本身不能 `read`，但会出现在 `/proc/<pid>/fd/<pathfd>`。这个路径以 `/proc` 开头，既不包含点号，也不含 `self`，恰好通过审查。先从程序写入的 `/tmp/log` 读取 shellcode 子进程 PID，再在 shellcode 中拼出：

```text
/proc/<pid>/fd/<pathfd>
```

完整系统调用链为：

```c
int pathfd = syscall(SYS_open_tree, AT_FDCWD, "/flag2", 0);
int fd = syscall(SYS_open, proc_fd_path, O_RDONLY, 0);
ssize_t n = syscall(SYS_read, fd, buffer, sizeof(buffer));
syscall(SYS_write, 1, buffer, n);
```

第二次 `open` 会进入 user-notification 处理器。高权限父进程跟随 `/proc/<pid>/fd/<pathfd>` 指向 `/flag2`，把一个真正可读的 FD 注入低权限子进程，随后即可读取并输出第二段 flag。

比赛发布版本还存在 `/sbin` 文件归属和 `/flag2` 权限设置错误，能产生更简单的非预期解；上述流程是源码和官方题解对应的预期利用链。

## 方法总结

“不可读文件”仍可能通过执行映射和调试接口泄露；“只允许 `/proc`”的路径过滤也不等于只能访问无害对象，因为 `/proc/<pid>/fd` 是指向任意已打开路径的能力入口。本题两问分别利用 ptrace 的寄存器控制能力，以及 `open_tree`、procfs FD 链接和 seccomp FD 注入之间的组合。设计沙箱时应基于最小系统调用集合和真实对象权限，而不是字符串前缀过滤。
