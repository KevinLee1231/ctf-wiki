# r3map

## 题目简述

r3map 是一道 Linux 内核利用题。每次连接先完成 SHA-256 PoW，再启动一台带 KASLR、PTI、SMEP 和 SMAP 的 QEMU 虚拟机。玩家位于 `nsjail` 创建的用户、挂载、PID 等命名空间中，只能访问以 `0660` 权限暴露的 `/dev/r3map_dev`。

真正的 flag 不在初始内存盘里的诱饵 `/flag` 中。`run.sh` 通过只读 9p 共享把宿主侧动态 flag 挂载为 `r3flag`，`/init` 再将其挂到：

```text
/root/flagfs/flag
```

`/root` 和 `/root/flagfs` 权限均为 `0700`，而 nsjail 还隔离了挂载命名空间。因此仅把当前进程的 UID 改成 0 并不足以完成题目，还需要回到初始挂载命名空间，或者等价地替换当前进程的文件系统视图。

漏洞位于内核模块 `r3map.ko`。模块提供 `MEM` 和 `VBO` 两类对象：`MEM` 负责持有页，`VBO` 是映射到 `MEM` 的虚拟缓冲区。并发执行 bind 与 unbind 时存在生命周期竞争，可让 `VBO` 留下指向已释放页的陈旧引用。后续把该页回收为当前进程的页表页，即可把 UAF 提升为任意物理内存读写。

## 解题过程

### 1. 确认部署与隔离边界

QEMU 启动参数中启用了 KASLR、PTI、SMEP 和 SMAP，并通过如下 9p 设备提供 flag：

```sh
-virtfs "local,path=$FLAG_SHARE_DIR,mount_tag=r3flag,security_model=none,readonly=on"
```

initramfs 中的 `/init` 执行：

```sh
mount -t 9p -o trans=virtio,version=9p2000.L,ro r3flag /root/flagfs
chmod 0700 /root /root/flagfs
insmod /lib/modules/r3map.ko
chown 0:1000 /dev/r3map_dev
chmod 0660 /dev/r3map_dev
```

随后以 `ctf` 用户启动 nsjail。配置中开启 `clone_newuser`、`clone_newns`、`clone_newpid` 等命名空间，并禁止 `mount`、`setns`、`unshare`、`userfaultfd`、`io_uring`、`bpf` 等常见利用辅助系统调用。

这两点决定了最终利用必须同时解决：

1. 内核权限提升；
2. 从 jail 的文件系统与挂载视图切换回 init 的视图。

### 2. 建立驱动对象与触发竞态

利用中使用的 ioctl 及主要参数结构如下：

```c
#define IO_ALLOC 0xc0186b01UL
#define IO_FREE  0x40086b02UL
#define IO_MAP   0x40306b03UL
#define IO_RUN   0x40286b04UL

struct alloc_req {
    uint32_t type;
    uint32_t attr;
    uint64_t size;
    uint64_t id;
};

struct map_req {
    uint64_t vbo;
    uint64_t mem;
    uint64_t voff;
    uint64_t len;
    uint64_t moff;
    uint64_t cmd;
};

struct run_req {
    uint64_t id;
    uint64_t off;
    uint64_t len;
    uint64_t user;
    uint32_t op;
    uint32_t reserved;
};
```

先分配一个页大小的 `MEM` 和一个 `VBO`，在 `MEM` 页中写入容易识别的标记。两个线程同时对同一组对象执行 bind 与 unbind：

```text
线程 A：bind VBO -> MEM
线程 B：unbind VBO
```

竞态成功时，驱动的逻辑状态认为映射已经解除，但通过 `VBO` 仍能读到 `MEM` 中的标记。这说明 VBO 内部残留了引用。此后释放 `MEM` 对象，便得到一个仍能通过旧 VBO 访问的释放后页。

### 3. 迫使 shrinker 真正归还页面

模块并不会立即把释放的 `MEM` 页交回伙伴分配器，而是暂存在自己的缓存链表中。对 `r3map.ko` 的符号和字符串检查可见：

```text
r3map_mem_cache
r3map_mem_cache_pages
r3map_mem_cache_shrinker
r3map_mem_cache_scan
r3map: shrinker freed %lu cached pages
```

因此只调用 `IO_FREE` 不足以让其他内核子系统复用该页。利用需要 fork 一个子进程，分配并实际触碰大量内存，制造回收压力，促使 r3map 的 shrinker 把缓存页返还给系统。压力子进程随后被 OOM 杀死并不影响利用，它只负责触发页面回收。

### 4. 把陈旧页回收为 PTE 页

下一步要让内核把刚释放的 4 KiB 页用作当前进程的页表页。先禁用透明大页：

```c
prctl(PR_SET_THP_DISABLE, 1, 0, 0, 0);
```

再预留大量按 2 MiB 对齐的匿名 VMA，并逐页访问，迫使内核分配大量普通 PTE 页。与此同时不断经旧 VBO 读取陈旧页，检测内容是否呈现页表特征：大量 64 位项具有相似的低位权限标志，而物理页号随映射变化。

一旦命中，旧 VBO 实际上就能改写当前进程的一张 PTE 表。一张 4 KiB PTE 页有 512 个 8 字节表项，能够控制连续：

```text
512 * 0x1000 = 0x200000
```

即 2 MiB 用户虚拟地址窗口。

### 5. 构造任意物理内存读写

保留原 PTE 的有效位、用户位、可写位和 NX 等标志，只替换物理页框号：

```c
pte[i] = ((physical + i * 0x1000) & PFN_MASK) | saved_flags;
```

随后访问对应用户态窗口，便会读写指定的物理页。每次切换目标物理地址前还要使旧 TLB 项失效，否则 CPU 可能继续使用缓存的地址翻译。

这一原语把最初的单页 UAF 提升成了整个物理内存上的读写能力。

### 6. 定位 KASLR 偏移

利用扫描物理内存中的零结尾字符串：

```text
/sbin/modprobe\0
```

并筛选出属于 `modprobe_path` 的实例。将运行时物理地址减去从内核镜像中得到的静态物理地址，即可算出物理重定位偏移；结合 `init_cred`、`init_user_ns` 等已知符号关系，再得到内核虚拟地址偏移。

这里扫描 `modprobe_path` 的主要作用是寻找稳定的 KASLR 锚点。最终提权并不依赖触发未知二进制格式，也不需要真的通过 modprobe 执行用户态脚本。

### 7. 修补凭据获得完整 root 权限

当前 `ctf` 用户的 UID/GID 为 1000。利用在物理内存中搜索符合 `struct cred` 布局、各 UID/GID 字段均为 1000 的对象，并修补候选项：

- `uid`、`gid`、`euid`、`egid`、`suid`、`sgid`、`fsuid`、`fsgid` 设为 0；
- capability 位图设为全权限；
- 与用户命名空间相关的指针按 `init_cred`、`init_user_ns` 修正。

为避免只命中 `real_cred` 或某个线程副本，可以对合理候选逐一修补，并在用户态用 `getuid()`、`geteuid()` 验证。

### 8. 修补 `task_struct` 逃离挂载命名空间

UID 变为 0 后仍看不到 `/root/flagfs/flag`，因为当前进程仍处于 nsjail 创建的挂载命名空间。先设置一个独特的进程名：

```c
prctl(PR_SET_NAME, "exploit_self", 0, 0, 0);
```

再扫描物理内存，按 `comm` 字段定位当前 `task_struct`。该内核版本中公开利用使用的相关偏移为：

```text
task_struct->real_cred : 0x750
task_struct->cred      : 0x758
task_struct->comm      : 0x768
task_struct->fs        : 0x798
task_struct->nsproxy   : 0x7b0
```

将当前任务的：

```text
task->fs      = init_fs
task->nsproxy = init_nsproxy
```

即可继承 init 进程的根目录、当前工作目录和命名空间集合。此时直接打开：

```text
/root/flagfs/flag
```

即可得到：

```text
r3ctf{MInd_tH3-G4p_6tw_61ND_4ND-Un61nd134f8bfe}
```

完整的竞态、物理扫描和远程上传代码可参考公开题解：[hax1ng 的 r3map writeup](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/r3map.md)。上文已经保留漏洞成因、页表回收策略、KASLR 定位以及命名空间逃逸的全部关键逻辑，外链主要用于保存与该内核构建绑定的完整 exploit 常量。

## 方法总结

完整利用链为：

```text
bind/unbind 竞态
  -> VBO 残留对 MEM 页的引用
  -> 释放 MEM 并制造内存压力
  -> shrinker 归还陈旧页
  -> 大量分配 PTE 页完成回收
  -> 经旧 VBO 改写当前页表
  -> 任意物理内存读写
  -> 扫描 modprobe_path 计算 KASLR 偏移
  -> 修补 cred 获得 root 与 capabilities
  -> 修补 task_struct 的 fs/nsproxy
  -> 读取初始挂载命名空间中的 9p flag
```

本题最值得保留的两个判断是：

1. 释放对象不等于页面已经回到通用分配器。出现 shrinker 或私有页缓存时，必须先验证回收路径，再安排 reclaim；
2. 内核态把 UID 改成 0 不必然等于逃离容器或 jail。若目标文件存在于另一个挂载命名空间，还要检查 `task_struct` 的 `fs` 和 `nsproxy`，或者寻找其他能进入宿主视图的等价路径。
