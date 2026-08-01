# d3kheap2pro-revenge

## 题目简述

`d3kheap2pro-revenge` 修复了普通版中两条绕过内核模块的非预期路径：通过 OOM Killer 进入 BusyBox 的无认证 root shell，以及利用 QEMU TCG 跨页访问检查错误取得任意物理内存写。模块自身的漏洞没有改变，仍然要从自定义 `kmem_cache` 上的 double free 出发，利用 Linux 7.1 引入的 per-CPU sheaf 完成 cross-cache attack。

模块为每个 slot 保存对象指针和引用计数。对象创建后引用计数从 1 又被递增到 2；释放操作每次都会递减计数并调用 `kmem_cache_free()`，但不会清空指针：

```c
atomic_dec(&slot->ref_count);
kmem_cache_free(d3kheap2pro_cachep, slot->buffer);
```

因此，同一个 slot 连续释放两次会把同一对象地址提交给 SLUB 两次。由于模块没有实现对象内容的读写接口，利用重点不在篡改对象本身，而在控制这个重复地址经过 sheaf、slab、PCP 和 buddy system 的回收时序。

## 解题过程

### 1. 把 double free 延迟到 sheaf flush

Linux 7.1 在传统 SLUB 路径前增加了 per-CPU sheaf。快速释放时，对象地址先进入 main sheaf；main 满后会与 spare 交换，继续积累时再与 node barn 交互。只有 sheaf 被 flush，内部的批量释放逻辑才真正重建 slab freelist 并更新 `inuse`。

这层延迟允许先把重复引用分散到不同批次，再精确控制它们何时回到 slab：

```text
重复对象地址进入不同 sheaf
  → 填充 main、spare 与 barn
  → 触发批量 flush，使目标 slab 的 inuse 归零
  → 页面经过 PCP 回到 buddy system
  → 由其它 cache 重占同一物理页
```

同一个批次中不能直接放入相邻的重复地址，否则 hardened freelist 可能形成自环并触发内核异常。布局时需要同时考虑 sheaf 容量、每个 slab 的对象数以及不同 CPU 上的缓存状态，用不同对象填充各批次，并保留相邻页中的活对象以降低 buddy 合并对布局的干扰。

### 2. 让 credential 对象重占目标页

目标 slab 页被完整回收后，大量创建 credential 相关对象，使该页以新的 cache 身份重新分配。为了提高命中率，可以预热分配器、用多个存活的 credential 作为哨兵控制页面密度，再反复驱动目标页在原 cache 与 credential cache 之间流转。

这里必须区分公开的 `kmem_cache_free()` 与 sheaf 内部的延迟批量释放。页面已经由其它 cache 重占后，再从模块接口公开释放旧指针会触发 cache 一致性检查；预先留在 sheaf 中的 stale 指针则会在后续 flush 时进入内部批量路径，从而把已重占页面上的对象再次交给分配器。

### 3. 把 `INIT_ON_ALLOC` 变成写零原语

内核启用了 `CONFIG_INIT_ON_ALLOC_DEFAULT_ON`。新对象分配时的初始化清零原本用于防止旧数据泄露，但当回收页被 credential 对象重占时，它也会把 `cred` 中的 uid、gid、euid、egid、fsuid 和 fsgid 等字段清零，等价于获得 root 身份字段。

不能在此后直接调用 `setresuid()`：同一轮清零还会破坏 `user_ns`、`user`、`ucounts` 等指针，而这类权限变更路径可能继续解引用它们并导致 kernel panic。可改用只依赖 `cred->fsuid` 的检查路径，例如调用 `fchmodat2()` 修改 root 所有文件的权限，再读取 flag。利用进程应在完成目标操作后避免触发会回收已损坏 credential 的正常退出路径。

### 4. revenge 版修复的旁路

普通版 BusyBox 的启动配置会在前置进程退出后进入 `askfirst:/bin/ash`。通过持续制造内存压力触发 OOM Killer，有机会让 init 直接在控制台拉起 root shell。revenge 版修正了这一启动配置，因此资源耗尽不再等价于提权。

另一条旁路位于 QEMU TCG 的 `access_ptr()`。旧实现先计算 `size - len`，特殊的跨页访问可以让该减法发生无符号下溢并取得越界 host 指针。修复后先验证页内偏移，再检查：

```c
len <= size - offset
```

同时还会确认跨页访问的第二个 host buffer 存在。这样切断了 QEMU 任意读写，但不会影响模块中的引用计数错误、sheaf 延迟释放和 credential 重占，因此预期的 cross-cache 利用链仍然成立。

## 方法总结

- 核心原语是自定义 `kmem_cache` 上的 double free。per-CPU sheaf 并没有消除漏洞，而是把真正的 free 延迟到了可操纵的批处理时机。
- 页面回收需要同时考虑 sheaf、slab、PCP 与 buddy system；页面由 credential cache 重占后，利用 `INIT_ON_ALLOC` 的清零副作用把身份字段变成 root。
- 清零后的 `cred` 含有损坏指针，不能照搬常规 `setresuid()` 提权。选择只检查 `fsuid` 的 `fchmodat2()` 路径，能够避开对 `user_ns` 等字段的继续访问。
- revenge 版只修复 OOM shell 与 QEMU TCG 越界两条非预期路径，不能再绕过模块本身的 allocator 利用。
