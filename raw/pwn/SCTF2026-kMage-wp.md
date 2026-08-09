# kMage

## 题目简述

内核模块注册 `/dev/sycmem`，维护最多 `0x400` 个、每个 `0x1000` 字节的专用 slab 对象。`SYCMEM_FREE` 先 `kmem_cache_free()`，随后才通过用户提供的 `sync` 地址执行 `copy_to_user`，最后才清空 slot 指针；与此同时 `READ/WRITE` 既不加锁，也不检查 `slot->freeing`。

只要把 free 线程阻塞在 `copy_to_user`，另一个线程就能经未清空的 slot 使用已释放对象。由于模块在 freeing 期间禁止再次 `ALLOC`，必须让 slab 页回到 buddy，再由其他内核子系统跨 cache 回收。本题选择 pipe ring 的 `pipe_buffer` 数组完成 reclaim。

## 解题过程

### 1. 用 PunchHole 放大 UAF 窗口

创建 memfd 并扩展为大块 shmem，将 `sync` 指向其中一页。辅助进程对同一 memfd 建立大量 alias，并循环执行：

```c
fallocate(fd,
    FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
    0, punch_size);
```

当 free 线程的 `copy_to_user(sync, &byte, 1)` 碰上被 punch 的页，会进入 shmem fault/重建路径而长时间停住。此时对象已经释放，`slot->ptr` 和 `slot->size` 却仍可被无锁的 READ/WRITE 读取，形成稳定 dangling pointer。

### 2. 用 pipe_buffer 跨 cache 回收

先创建约 192 个 pipe，并向每个 pipe 写入不同长度 `0x80+i` 作为标签。填满 sycmem 的 `0x400` 个 slot，选择中间一批 victim，释放其余对象并对 victim 启动阻塞 free，使专用 cache 的空 slab 能归还 buddy。

随后对所有 pipe 调用较大的 `F_SETPIPE_SZ`。扩容分配接近一页大小的 `pipe_buffer` 数组，有机会占用刚释放的物理页。通过 stale `SYCMEM_READ` 扫描 victim 页，按以下特征识别命中项：

```text
page    是 vmemmap 区的 struct page 指针
ops     是内核地址，指向 anon_pipe_buf_ops
offset  为 0
private 为 0
len     等于某个 0x80+i 标签
```

由 `len-0x80` 可定位具体 pipe，记录该 entry 在 stale 页中的偏移。

### 3. 构造内核任意读

泄漏的 `ops` 与静态 `anon_pipe_buf_ops` 符号之差给出 KASLR slide；`page` 指针则用于推断 vmemmap 基址。经 stale WRITE 临时改写命中 entry：

```c
pb.page   = target_page;
pb.offset = target_offset;
pb.len    = wanted_length;
pb.ops    = original_ops;
pb.flags  = 0;
```

再用 `tee()` 把该 pipe buffer 复制到临时 pipe，用户态 `read()` 即可取得目标物理页内容。每次读取后恢复原始 entry，避免关闭 pipe 时因伪造字段崩溃。

`modprobe_path` 的运行时虚拟地址可由 slide 算出，但 pipe buffer 需要 `struct page *`。官方利用在可能的内核物理加载区间内按 2 MiB 枚举，把 `modprobe_path-_text` 的偏移换算为候选物理页，再用任意读检查目标位置是否包含 `/sbin/modprobe`，从而确定正确页和页内偏移。

### 4. CAN_MERGE 写 modprobe_path

把命中 entry 改为目标页，令 `offset=target_off-1`、`len=1`，并设置 `PIPE_BUF_FLAG_CAN_MERGE`。向 pipe 写入 `/home/ctf/x\0` 时，新数据会从 `offset+len` 开始追加，正好覆盖 `modprobe_path`。

预先写好 `/home/ctf/x`，内容负责把 `/flag` 复制到普通用户可读位置并修改权限。随后尝试未注册协议族的 socket，触发 `request_module()`；内核以 root 调用新的 helper，最后读取复制出的 flag。完整实现位于官方 `exploit.c`。

## 方法总结

本题利用的决定性步骤是“释放已完成、slot 清理未完成”的并发窗口。PunchHole 不是漏洞本身，而是把瞬时竞态变成可操作窗口；cross-cache 则绕过了模块禁止同 cache 重分配的限制。后半段应保持 `pipe_buffer.ops` 不变，只围绕 `page/offset/len/flags` 建立可恢复的读写原语，并用 `/sbin/modprobe` 字符串验证物理页定位后再写入，避免盲写导致内核 panic。
