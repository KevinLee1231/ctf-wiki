# dekatron

## 题目简述

菜单程序删除对象时释放其缓冲区，却没有清空数组中的指针，形成 use-after-free。可利用释放后读取泄露 libc，再通过 tcache 污染覆盖 `__free_hook`。

## 解题过程

先申请一个足够大的块并释放，使其进入 unsorted bin；通过仍然存在的悬空指针读取块内的 arena 指针，计算 libc 基址。随后布置同尺寸 tcache 块，在 UAF 写入阶段把空闲链表的 `fd` 改为 `__free_hook`，连续两次申请后即可获得指向 hook 的块：

```python
libc.address = arena_leak - arena_offset
target = libc.symbols["__free_hook"]

# 释放同尺寸块后，经悬空指针把 tcache fd 改为 target
edit_freed_chunk(p64(target))
alloc(b"/bin/sh\x00")
alloc(p64(libc.symbols["system"]))
```

最后释放保存 `/bin/sh` 的块，实际调用 `system("/bin/sh")`，读取：

```text
n00bz{use_after_free_for_the_winn}
```

## 方法总结

释放对象后必须同时清除所有别名指针。本题利用链依赖两个阶段：unsorted bin 指针提供地址基准，tcache 链表提供任意地址分配；任一泄露或堆布局未校验都会使最终 hook 覆盖不可靠。
