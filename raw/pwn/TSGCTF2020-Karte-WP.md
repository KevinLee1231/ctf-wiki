# TSGCTF2020 Karte WP

## 题目简述

程序用 `realloc` 管理最多 32 个 `Vec`，每个对象的用户区开头依次保存 `size` 和 `id`：

```c
typedef struct {
    unsigned long long size;
    unsigned long long id;
    unsigned long long data[];
} __attribute__((__packed__)) Vec;
```

释放函数只执行 `realloc(vecs[num], 0)`，却没有把 `vecs[num]` 清零。数组中因此保留悬空指针，`show` 和 `change_id` 仍可读取、修改已释放块。最终目标不是劫持控制流，而是让 BSS 上的全局变量 `authorized` 变为非零；菜单项 5 随后会直接执行 `system("/bin/sh")`。

## 解题过程

题目使用 glibc 2.31。`Vec` 的字段布局恰好与空闲链表元数据重叠：释放后，偏移 0 的 `size` 会被 fastbin/tcache 的链指针覆盖，偏移 8 的 `id` 在 fastbin 情形下仍可作为对象标识。先申请 9 个 `0x50` 对象，释放 6 个填充 tcache，再按 `1、0、2` 的顺序释放其余对象，并重新申请一次。此时通过仍然可用的编号 2 调用 `show`，读出的所谓 `size` 实际是堆链指针：

```python
_, leak = show(2)
heap_base = leak - 0x350
```

对 `0x90` 大小重复相似布局。先填满对应 tcache，使额外释放的块进入 unsorted bin；再申请 `0xa0`，迫使 allocator 扫描 unsorted bin，并把合适的 `0xa0` chunk 归入 smallbin。已释放 `Vec` 的 `id` 字段此时与 smallbin 的 `bk` 重叠，因此可从已知堆地址定位记录，并用 `show` 读出 `main_arena` 指针：

```python
_, arena = show(heap_base + 0x7f0)
libc_base = arena - 0x1ebc70
```

利用的核心是 glibc 的 tcache stashing 行为。malloc 从 smallbin 取块时，会把同尺寸剩余节点转存到未满的 tcache，并执行近似以下链表更新：

```c
bck = tc_victim->bk;
bin->bk = bck;
bck->fd = bin;
tcache_put(tc_victim, tc_idx);
```

程序启动时先在 `name` 中布置伪造链表指针，以满足 smallbin 的一致性检查；目标地址经过偏移调整，使 `bck->fd = bin` 最终写到 `authorized` 附近。随后利用悬空对象调用：

```python
change_id(libc_base + 0x1ebc70, victim)
extend(100, 0x90)
```

第一步搜索 `id == main_arena` 的已释放对象，并把与 smallbin `bk` 重叠的字段改成 `victim`。第二步把原来的 `0x50` 对象扩展为 `0x90`，触发对应 smallbin 分配及剩余节点的 tcache stashing。链表维护把非零的 bin 指针写入 `authorized`，之后选择菜单项 5 即可得到 shell。仓库中的 flag 为：

```text
TSGCTF{Realloc_is_all_you_need~}
```

## 方法总结

本题利用链由 `realloc(ptr, 0)` 后未清理指针造成的 UAF 驱动。先把释放块字段解释为空闲链表指针以泄露 heap 和 libc，再利用 `id` 位于偏移 8、正好覆盖 smallbin `bk` 的布局实施 tcache stashing unlink 写，最终只需把 `authorized` 改成任意非零值。防御上，释放后必须立即清空所有别名并禁止继续按旧 `id` 查找；同时不要让用户可修改的业务字段与 allocator 元数据在 UAF 状态下形成可控重叠。
