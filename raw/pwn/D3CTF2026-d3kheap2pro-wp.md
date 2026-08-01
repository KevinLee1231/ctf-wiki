# d3kheap2pro

## 题目简述

题目在 Linux 7.1.4 中加载 `d3kheap2pro.ko`。普通用户可以通过 `/proc/d3kheap2pro` 操作一个独立的 `kmem_cache`，但模块只实现对象分配与释放，没有可直接利用的对象读写接口。内核同时开启 KASLR、SMEP、SMAP、KPTI、SLUB freelist hardening、page allocator shuffle 以及 `INIT_ON_ALLOC`、`INIT_ON_FREE`。

漏洞来自引用计数初始化错误。对象创建后先把引用计数设为 1，又执行一次 `atomic_inc()`，使初始值变成 2；释放路径递减计数并调用 `kmem_cache_free()`，却不清空对象指针。因此同一个 slot 可以连续释放两次，把同一地址提交给 SLUB 两次。

模块没有提供 edit/show 功能，不能把 double free 直接转换为函数指针劫持。预期利用需要操纵 Linux 7.1 新引入的 per-CPU sheaf，完成 cross-cache attack，再借助 `INIT_ON_ALLOC` 的初始化清零把 credential 身份字段变成 root。

## 解题过程

### 1. 确认 double free

分配路径中的关键代码为：

```c
slot->buffer = kmem_cache_alloc(
    d3kheap2pro_cachep,
    GFP_KERNEL | __GFP_ZERO
);
atomic_set(&slot->ref_count, 1);
atomic_inc(&slot->ref_count);
```

释放路径只检查计数是否大于 0：

```c
if (atomic_read(&slot->ref_count) <= 0)
    return -EPERM;

atomic_dec(&slot->ref_count);
kmem_cache_free(d3kheap2pro_cachep, slot->buffer);
```

第一次释放后引用计数为 1，第二次释放后为 0，但两次 `kmem_cache_free()` 使用的是同一个地址。由于 `slot->buffer` 始终保留，double free 可以稳定触发。

### 2. 为什么旧式 cross-cache 路径失效

旧内核中常见的做法是先把自定义 slab 页归还 buddy system，用其它 cache 重占，再从模块接口释放旧指针形成 UAF。Linux 7.1 的公开 `kmem_cache_free()` 会检查对象所在页的 `slab_cache`：

```c
slab = virt_to_slab(x);
if (unlikely(!slab || slab->slab_cache != s)) {
    warn_free_bad_obj(s, x);
    return;
}
```

一旦页面已经属于另一种 cache，再以 `d3kheap2pro_cachep` 调用公开释放入口只会触发告警，不会完成释放。因此必须在页面仍属于原 cache 时让检查通过，同时把真正修改 slab 状态的动作延迟到页面重占之后。

### 3. 利用 per-CPU sheaf 延迟释放

Linux 7.1 在传统 SLUB freelist 之前增加了 per-CPU sheaf。每个 CPU 为每个 cache 保存容量为 12 的 main sheaf 和 spare sheaf，NUMA node 上还有可保存 10 个满 sheaf 的 barn。快速释放时，对象地址先被追加到 main 的指针数组；只有 sheaf 被驱逐或 flush，内部批量释放才真正重建 freelist 并扣减 `slab->inuse`。

利用链据此分成两阶段：

```text
页面仍属于 d3kheap2pro_cache
  → 把两份对象引用分散到不同 CPU、不同 sheaf 批次
  → 填满 main、spare 与 barn，控制批次进入待 flush 状态
  → flush 第一组地址，使目标 slab 的 inuse 归零
  → 页面经过 PCP 回到 buddy system
  → credential cache 重占同一物理页
  → 再 flush 预先保存的 stale 地址
```

这里有两个重要约束：

1. 同一次 bulk free 中不能出现重复地址，否则 hardened freelist 可能把对象链接到自身并触发 `BUG()`；每个批次都要由不同对象组成。
2. 页面回收不仅经过 slab 与 buddy system，还要经过 per-CPU page set。需要预热并清理相关 CPU 的分配器状态，同时保留相邻 slab 的活对象，避免目标页与 buddy 合并成更高 order 后无法被预期分配重占。

### 4. 用 credential 重占页面

目标 slab 页回到 buddy system 后，创建大量 credential 对象争夺该页。可使用多个长期存活的 helper 进程作为 credential 哨兵，并反复分配、回收其它 credential，控制页面密度和回收节奏。命中后，旧 sheaf 中的 stale 地址仍指向这页；后续批量 flush 会把已经属于 credential cache 的对象再次交给分配器，完成 cross-cache 重叠。

为了避免在身份字段损坏后再做复杂路径解析，应在触发重占前打开目标文件，保留可供 `AT_EMPTY_PATH` 使用的文件描述符。攻击进程还应尽量避免走会释放已损坏 credential 的正常退出路径。

### 5. 将 `INIT_ON_ALLOC` 变成写零原语

`CONFIG_INIT_ON_ALLOC_DEFAULT_ON` 会在新对象分配时清零内存。该保护原本用于阻止旧数据泄露，但 credential 对象落到已操纵页面时，初始化过程也会把 uid、gid、euid、egid、fsuid 和 fsgid 等身份字段清零，等价于获得 root 权限字段。

整块清零还会破坏 `user_ns`、`user`、`ucounts` 等指针，所以不能直接调用 `setresuid()`：后续权限校验可能解引用空指针并导致 kernel panic。更稳妥的做法是把 `fchmodat2()` 作为命中后的第一个新系统调用，配合预先打开的 fd 和 `AT_EMPTY_PATH` 修改 root 文件权限。该检查只需要 `cred->fsuid`，不会继续访问已清零的 namespace 指针。随后即可读取 `/flag`；也可以修改 BusyBox 等 root 执行文件的权限，再通过正常启动链取得高权限执行。

### 6. 普通版的两条非预期路径

普通版还存在两条绕过模块漏洞的提权方式。

第一条来自 BusyBox 的启动配置：

```text
::askfirst:/bin/ash
```

持续制造内存压力触发 OOM Killer 后，阻塞启动流程的低权限进程可能被杀死，BusyBox init 随即继续执行 `askfirst` 并在控制台启动 root shell。这个问题的本质是低权限用户可控的 OOM 事件被错误地连接到无认证的高权限 shell；[CTF Wiki 的 OOM shell 条目](https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/tricks/out-of-mem-shell/)整理了同类配置问题。

第二条来自 QEMU TCG 的 `access_ptr()` 边界检查。旧实现先计算 `size - len`，特殊的跨页访问可触发无符号下溢并获得越界 host 指针，再结合页表喷射形成任意物理内存读写。该路径属于虚拟化平台漏洞，与模块的 double free 无关。

revenge 版本修复了 BusyBox 启动配置和 QEMU 边界检查，但模块漏洞、sheaf 延迟释放与 credential 清零链保持不变。

## 方法总结

- 漏洞起点是独立 `kmem_cache` 上的 double free，但 Linux 7.1 的 cache 一致性检查使“页面重占后再公开 free”的旧套路失效。
- per-CPU sheaf 把地址入队与真正更新 slab 状态分离。先让旧指针通过检查，再把 flush 延迟到页面被 credential cache 重占之后，即可恢复 cross-cache 利用能力。
- `INIT_ON_ALLOC` 不只是保护机制，也能成为可控写零原语。清零 credential 身份字段后，应选用只依赖 `fsuid` 的 `fchmodat2()`，避开对已损坏 namespace 指针的访问。
- OOM shell 与 QEMU TCG 越界属于普通版的非预期路径，不能和内核模块的 allocator 利用混为一谈。
