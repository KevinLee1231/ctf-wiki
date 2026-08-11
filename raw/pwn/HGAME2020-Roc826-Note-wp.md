# Roc826's_Note

## 题目简述

这是一个菜单式堆题。删除笔记后指针没有清空，因而同时存在 UAF 和 double free。利用路线是：先从 unsorted bin 泄露 libc，再对 0x58 大小的 fastbin 做重复释放，伪造链表指针覆盖 GOT，最后把 `free` 改成 `system` 并释放保存 `/bin/sh` 的块。

## 解题过程

先申请一个 0x80 大小的块和三个 0x58 大小的块，最后一个块写入：

```text
/bin/sh\x00
```

释放 0x80 块后再通过 UAF 的 show 功能读取其内容。该块进入 unsorted bin，链表指针落在 `main_arena` 附近，因此可计算：

```python
libc_base = leak - libc.sym["__malloc_hook"] - 0x68
system = libc_base + libc.sym["system"]
```

`0x68` 是题目所用 libc 的结构偏移，换 libc 后必须重新核对。接着对两个 0x58 块执行：

```text
free(1)
free(2)
free(1)
```

形成 `A -> B -> A` 的 fastbin double-free 链。下一次申请 A 时，把其 `fd` 改为 `0x601ffa`；连续取回链中块后，分配结果会落到 GOT 表附近。之所以选择 GOT 前 6 字节，是为了同时满足 16 字节对齐检查，并让真正的函数指针位于写入数据偏移 14 的位置：

```python
payload = b"A" * 14 + p64(system)[:6]
```

写入后，目标 GOT 项从 `free` 变为 `system`。最后删除先前保存 `/bin/sh\x00` 的笔记，程序实际执行：

```c
system("/bin/sh");
```

获得 shell 后读取 flag。

## 方法总结

- 核心技巧：unsorted bin 泄露 libc，fastbin double free 控制下一次分配位置，再用 GOT overwrite 把释放操作变成命令执行。
- 识别信号：free 后索引仍可 show/delete，是 UAF 与 double free 的典型入口。
- 复用要点：fastbin 目标必须满足版本相关的尺寸、对齐和链表检查；`0x601ffa`、`0x68` 等常量都属于题目构建，迁移时应由 ELF 与 libc 符号重新推导。
