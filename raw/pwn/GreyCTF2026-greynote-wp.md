# greynote

## 题目简述

`/dev/greynote` 提供分配、释放、编辑和读取 0x400 字节 note 的 ioctl。释放函数调用 `kmem_cache_free()` 后没有清空全局槽位，留下完整 UAF 读写。内核启用 SMEP、SMAP、KASLR，并使用带 percpu sheaves 的 Linux 7.0.x SLUB；note 还位于 `SLAB_NO_MERGE` 专用 cache，无法直接用普通 `kmalloc-1024` 对象同 cache 重占。

利用需要把整个 note slab 的物理页送回 buddy allocator，再让该页被复用为页表，形成跨 cache Dirty Pagetable，最后扫描物理内存并修改当前进程的 `cred`。

## 解题过程

驱动漏洞等价于：

```c
kmem_cache_free(note_cache, notes[idx]);
/* notes[idx] 仍保留旧地址 */
```

之后 `GN_READ` 与 `GN_EDIT` 仍会解引用该指针。每个 note 为 1024 字节，一个 slab 含 16 个对象，因此底层是 16 KiB 的 order-2 页块。

新内核的 free 会先进入当前 CPU 的 sheaf，再溢出到 per-node barn，只有前端缓存都满后才真正归还 slab。首先把进程固定在 CPU 0，分配 `NGROOM=2048` 个 note，再全部释放。这个数量足以淹没 sheaf/barn 储备、排空多个完整 slab，并让 order-2 页块回到 buddy；所有索引仍是 UAF handle。

接着以 `MAP_NORESERVE` 建立 1 GiB 匿名映射并向上对齐到 2 MiB。每次 fault 一个完整 2 MiB 区间的 512 个页面，会分配一个 order-0 PTE 页，同时 buddy 可能拆分刚释放的 order-2 块。每 fault 一段后读取全部悬空 note，寻找密集的用户态 PTE：

```c
if ((pte & 1) && (pte & 4) && PFN(pte) != 0)
    present++;
```

当 128 项中超过 64 项形似 `0x8000_0000_xxxxx_067`，说明某个 1 KiB note 已覆盖活跃 PTE 页的一部分。记录其索引 `U` 和相应 2 MiB 虚拟区间 `Rbase`。

一页有 512 个 PTE，而 note 只能覆盖连续 128 项，所以还需确定它对应四个 quarter 中的哪一个。对候选：

$$
Rbase+q\cdot128\cdot4096,\qquad q\in\{0,1,2,3\}
$$

分别对第一页执行 `MADV_DONTNEED`，再经 UAF 读取 PTE。只有正确 quarter 会让当前窗口的 `entry[0]` 改变，由此得到可控窗口基址 `V0`。

现在向 note 写入 128 个伪 PTE，即可把任意连续 128 个物理页映射到 `V0`：

```c
ptes[i] = ((pfn + i) << 12) | 0x267;
note_edit(U, ptes, 0x400);
```

`0x267` 除了 present、rw、user、accessed、dirty 外还包含 `_PAGE_SPECIAL`。这很关键：每轮用 `MADV_DONTNEED` 刷新 TLB 时，内核不会对攻击者伪映射的任意物理页执行 `put_page`，否则扫描会迅速破坏页引用计数并卡死内核。

按每轮 512 KiB 扫描物理地址 `0x01000000` 到 `0x20000000`，后者与 QEMU 的 512 MiB RAM 上限一致，避免读到设备 MMIO。当前进程 `cred` 的明显特征是一个较小的 `usage` 引用计数后紧跟八个值为 1000 的 32 位字段：

```text
uid, gid, suid, sgid, euid, egid, fsuid, fsgid
```

对匹配候选把八个字段清零，并立即检查 `getuid()==0`。命中当前进程的 `cred` 后执行 root shell，读取：

```text
grey{i sure hope you didn't cheese that challenge somehow U_U}
```

## 方法总结

专用 slab cache 阻断的是“对象级同 cache 重占”，但不能阻止物理页离开 slab 后被其他内核子系统使用。利用通过大量释放穿透 sheaves/barn，再用 PTE 页从 buddy 跨 cache 回收；UAF 因而变成页表编辑。2 MiB 对齐、quarter 定位、`_PAGE_SPECIAL` 和物理扫描上限都是稳定性条件，不是可省略的优化。最终采用数据型 `cred` 覆写，也绕开了 SMEP/SMAP/KASLR 下的控制流劫持难题。
