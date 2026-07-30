# d3kheap2pro-revenge

## 题目简述

这是 `d3kheap2pro` 修复两条非预期路径后的 revenge 版本。题目仍提供 Linux 7.1.4 内核、QEMU 启动脚本、rootfs 和一个名为
`d3kheap2pro.ko` 的内核模块。启动后普通用户可以读写
`/proc/d3kheap2pro`，目标是在开启 KASLR、SMEP、SMAP、KPTI 和
SLUB hardening 的环境中利用模块漏洞，最终读取只有 root 可读的
`/flag`。

官方 PDF 说明，revenge 修复的是普通版的 OOM root shell 与 QEMU TCG 非预期解，没有修改模块的预期 double free。对关键附件做只读差分也支持这一结论：内核、配置、启动参数、模块、BusyBox 和 `rcS` 均保持一致，来宾侧与非预期直接相关的变化是把 `/etc/inittab` 的 `askfirst` 从 root shell 改为 `poweroff`。宿主 QEMU 的补丁不在 qcow2 内，只能由官方说明和 TCG 修复代码确认。

附件中的关键保护如下：

| 项目 | 状态 |
| --- | --- |
| 内核 | Linux 7.1.4-a3infra-kernel |
| KASLR / KPTI | 开启 |
| SMEP / SMAP | 开启 |
| `CONFIG_SLAB_FREELIST_HARDENED` | 开启 |
| `CONFIG_SLAB_FREELIST_RANDOM` | 开启 |
| `CONFIG_INIT_ON_ALLOC_DEFAULT_ON` | 开启 |
| `CONFIG_INIT_ON_FREE_DEFAULT_ON` | 开启 |
| `CONFIG_KFENCE` | 开启，采样间隔 100 ms |
| `CONFIG_SHUFFLE_PAGE_ALLOCATOR` | 开启 |

本题与 2025 年的
[d3kheap2](https://github.com/arttnba3/D3CTF2025_d3kheap2)
使用了相似的模块漏洞，但不能直接套用旧利用。Linux 7.1 的
`kmem_cache_free()` 会检查待释放页的 `slab_cache`，旧版中把已经被
`kmalloc-cg-2k` 重占的对象再次交给自定义 cache 的做法会被拒绝。
本题的关键是利用 Linux 7.1 新的 per-CPU sheaf 批量回收路径，在指针
仍合法时把它保存进 sheaf，等页面被另一种对象重占后再由内部 bulk
free 路径释放。

revenge 与原版附件内置的本地测试 flag 相同：

```text
flag{arttnba3_t3s7_f1@9!}
```

整理本文时没有可用的 revenge 远端实例，也没有重新启动 QEMU 长时间复跑。下文利用在原版附件中已由 `ctf` 用户完整提权验证；其对 revenge 的可复用性来自关键二进制逐字节相同、官方明确只修复非预期路径，以及利用不依赖被修改的 `askfirst` 动作。本文不把本地测试 flag 冒充为 revenge 远程 flag。

## 解题过程

### revenge 差分与修复边界

顶层附件的 SHA-256 如下：

| 文件 | 原版与 revenge 的关系 | SHA-256 |
| --- | --- | --- |
| `bzImage` | 相同 | `04626cdcfed21ebdb422d247fee3603fd497cd63115f4780e96a7c3cd6d23089` |
| `kernel_config` | 相同 | `6969999fc52993d72a6d5debd428d10563f8fc64199b6dd7eda002d32f544f26` |
| `run.sh` | 相同 | `3f2dfd1e63adc8b7768446783f8583fc89ea2efa12ff9652086a115e0e52e15e` |
| `rootfs.qcow2` | 容器不同 | 原版 `36809dc7...`, revenge `8335a015...` |

只读展开两个 rootfs 后，预期利用依赖的关键文件为：

```text
/root/d3kheap2pro.ko
  7a91c020526fe3e0e7f0e7a23b4ba6d0fc42517ae0ecba782cb811b4ba465ead

/bin/busybox
  bbc4c150f0dd092062cda5430c6e795a8fb444a75fe74f61e847db2ac58634bf

/etc/init.d/rcS
  cb33be945f00de73d82c5a8604922e09bc5daabb55c402dd40365e05917c833e
```

三者在两个版本中完全相同。可见差异是：

```diff
 ::sysinit:/etc/init.d/rcS
-::askfirst:/bin/ash
+::askfirst:/sbin/poweroff
 ::ctrlaltdel:/sbin/reboot
```

这会切断“制造内存压力触发 OOM，再让 init 进入无认证 root shell”的旁路。官方还在宿主 QEMU 中修复了 `access_ptr()` 对 `size - len` 的无符号下溢，阻止 x87 跨页访问把 TCG 越界提升为任意物理内存读写。两处补丁分别位于来宾启动配置和宿主模拟器；模块 double free、Linux 7.1 sheaf 行为以及后续消息/管道利用均未变化。

### 模块逆向

模块维护 512 个 slot，每个 slot 的结构可以还原为：

```c
struct slot {
    void *buffer;
    atomic_t ref_count;
    uint32_t padding;
};

struct slot d3kheap2pro_bufs[512];
struct kmem_cache *d3kheap2pro_cachep;
```

初始化时创建大小为 `0x800` 的独立 cache：

```c
kmem_cache_create("d3kheap2pro_cache", 0x800, 0,
                  SLAB_PANIC | SLAB_ACCOUNT, NULL);
```

运行时 sysfs 给出的参数是：

```text
object_size    = 2048
objs_per_slab  = 16
order          = 3
sheaf_capacity = 12
```

两个有效 ioctl 为：

```c
#define IOCTL_ALLOC 0x3361626e
#define IOCTL_FREE  0x74747261
```

分配逻辑在成功分配对象后先把引用计数设成 1，随后又执行一次
`atomic_inc()`，因此初始引用计数为 2。释放逻辑只检查引用计数是否大于
0，随后无条件执行：

```c
atomic_dec(&slot->ref_count);
kmem_cache_free(d3kheap2pro_cachep, slot->buffer);
```

它没有把 `slot->buffer` 清零。于是同一个 slot 可以连续调用两次
`IOCTL_FREE`，同一地址会被提交给 SLUB 两次。

模块的 read/write 和另外两个 ioctl 都没有提供有效的对象内容读写能力，
所以这不是普通的“释放后直接改函数指针”。必须先把时间型 double free
提升成页级空间重叠，再借助内核中可由用户态读写的对象构造利用原语。

### 为什么 2025 年的利用不能直接使用

旧利用的主要过程是：

```text
自定义 slab 页
    -> buddy
    -> kmalloc-cg-2k 的 msg_msgseg
    -> 再通过旧模块指针 kmem_cache_free()
    -> msg_msgseg UAF
```

Linux 7.1 的公开释放入口增加了 cache 一致性检查：

```c
slab = virt_to_slab(x);

if (IS_ENABLED(CONFIG_SLAB_FREELIST_HARDENED) ||
    kmem_cache_debug_flags(s, SLAB_CONSISTENCY_CHECKS)) {
    if (unlikely(!slab || slab->slab_cache != s)) {
        warn_free_bad_obj(s, x);
        return;
    }
}
```

因此页面一旦属于 `kmalloc-cg-2k`，再执行
`kmem_cache_free(d3kheap2pro_cachep, x)` 只会产生
“object belongs to different cache”告警，不会完成释放。

本题内核的 SLUB 源码可参考
[Linux 7.1 `mm/slub.c`](https://github.com/torvalds/linux/blob/v7.1/mm/slub.c)。
与旧内核相比，普通对象的快速路径不再只依赖 slab 内的 freelist，而是
优先使用 per-CPU 的指针数组 sheaf。

### Linux 7.1 sheaf 的可利用行为

每个 CPU 对一个 cache 维护：

```text
main sheaf   12 个指针
spare sheaf  12 个指针
```

NUMA node 还维护一个 barn。本题中 barn 最多保存 10 个 full sheaf。

快速释放只把地址追加进当前 CPU 的 `main->objects[]`：

```c
pcs->main->objects[pcs->main->size++] = object;
```

此时不会修改 `slab->inuse`。只有 sheaf 被真正 flush 时，
`sheaf_flush_unused()` 才调用内部的 `__kmem_cache_free_bulk()`，
批量建立 freelist 并扣减 `inuse`。

当以下条件同时成立时：

```text
main 已满
spare 已满
barn 已有 10 个 full sheaf
```

下一次释放会走 `__pcs_replace_full_main()`，把 spare 交给
`sheaf_flush_unused()`。这给了我们一个延迟释放窗口：

1. 地址进入 sheaf 时，页面仍属于 `d3kheap2pro_cache`，公开入口检查通过；
2. 页面随后被归还 buddy 并由其他 cache 重占；
3. 更晚触发 sheaf flush，内部 bulk free 不再重新执行公开入口的
   `slab->slab_cache == s` 检查。

还有一个容易踩坑的细节：如果同一地址在一次 bulk free 中出现两次，
`build_detached_freelist()` 会尝试把对象链接到自身，
`SLAB_FREELIST_HARDENED` 会直接触发 `BUG()`。因此不能把 12 个相同地址
塞进一个 sheaf，必须让每个待 flush 的批次都由不同对象地址组成。

### 确定 36 次初始分配的真实页布局

cache 的 sheaf 容量为 12，但每个 order-3 slab 有 16 个对象。
`refill_objects()` 会先从 partial slab 取对象，不够 12 时再从新 slab
补齐；分配则从 sheaf 尾部按 LIFO 顺序弹出。

本地 GDB 验证得到前三次 refill 的 slot 布局：

```text
目标 slab A: slot 0..11, 20..23
buddy slab B: slot 12..19, 28..35
下一个 slab C: slot 24..27
```

形成该布局的过程是：

```text
第 1 次 refill: A 的 12 个对象
第 2 次 refill: A 剩余 4 个 + 新 slab B 的 8 个
                 LIFO 弹出后先得到 B[8]，再得到 A[4]
第 3 次 refill: B 剩余 8 个 + 新 slab C 的 4 个
                 LIFO 弹出后先得到 C[4]，再得到 B[8]
```

第三次 refill 不能省略。后面释放 A 时，B 中的 `slot 28..35` 仍保持
live，可以阻止 A 和物理相邻的 B 合并为 order-4。若 A+B 合并，
后续申请 order-3 的 `kmalloc-cg-2k` slab 不会立即拿到 A。

### 构造 CPU 0 的两个释放批次

先在 CPU 2/3 保存目标页 A 的第一份引用：

```text
CPU 2 main: A 的 slot 0..11，共 12 个不同地址
CPU 3 main: A 的 slot 20..23，共 4 个不同地址
```

随后在 CPU 0 释放每个对象的第二份引用，得到：

```text
CPU 0 spare: A[0..11]
CPU 0 main : A[20..23] + B[12..19]
```

为了让 CPU 0 的 spare 被真正 flush，在 CPU 1 先分配 144 个对象，再把
它们各释放一次：

```text
12 个 -> CPU 1 main
12 个 -> CPU 1 spare
120 个 -> barn 的 10 个 full sheaf
```

另外预分配 120 个 trigger 对象。先在 CPU 1 释放其中 73 个，会在第
1、13、25、37、49、61、73 次释放时 flush 七个普通 sheaf，从而建立
足够多的 partial slab，保证空 slab 会被真正还给 buddy，而不是留在
cache 的 `min_partial` 中。

最后切换到 CPU 0 再释放 13 个 trigger：

1. 第 1 次触发 flush `A[0..11]`，A 的 `inuse` 从 16 变成 4；
2. 之后 11 次填满新的 main；
3. 第 13 次 flush `A[20..23] + B[12..19]`；
4. A 的 `inuse` 从 4 变成 0，被归还 buddy；
5. B 仍有 `slot 28..35` 八个活对象，因此 A 保持为独立 order-3 块。

此时 CPU 2/3 仍分别保存 12 和 4 个指向已释放 A 页的地址。

### 第一次跨 cache 重占：victim msg_msgseg

大消息使用一个 `msg_msg` 头和一个 `msg_msgseg`：

```c
#define MSG_HEAD_DATA  (0x1000 - 0x30)
#define MSG_SEG_DATA   (0x800 - 8)
#define LARGE_MSG_DATA (MSG_HEAD_DATA + MSG_SEG_DATA)
```

`msg_msgseg` 的 8 字节 next 指针加上 `0x7f8` 字节数据，正好进入
`kmalloc-cg-2k`。

利用开始前先完成两种预处理：

1. 保留 480 个大消息，消耗现有 `kmalloc-cg-2k` 对象；
2. 为 victim、evil 和 attack 队列预分配只有头部的消息。

重占时先删除一个头部消息，再立即发送大消息。刚释放的头会被同一个
msg cache 重用，不会抢走 order-3 目标页；新产生的 2 KiB segment 则
用于重占 A。

喷射数使用 480，因为：

```text
480 % 12 == 0
480 % 16 == 0
```

既能清空当前 CPU 的 sheaf 状态，又能覆盖足够多的 order-3 候选。
这对启用了 `CONFIG_SHUFFLE_PAGE_ALLOCATOR` 的内核很重要。

### 用 stale sheaf 再次释放 victim slab

victim segment 重占 A 后，CPU 2/3 中保存的地址已经指向
`kmalloc-cg-2k` 对象。

CPU 2 的 main 有 12 个 stale 地址。再释放 13 个未使用的 trigger：

```text
第 1 个：把 stale main 移到 spare
后续 11 个：填满新 main
第 13 个：flush stale spare
```

这样先释放 A 页上的 12 个 victim segment。

CPU 3 的 main 里有剩余 4 个 stale 地址。先用 8 个合法 trigger 填满，
再以同样方式把它移动到 spare 并 flush。此批次包含：

```text
4 个 A 页上的 stale victim segment
8 个合法的 d3kheap2pro 对象
```

4 个目标地址与前一批的 12 个地址不同，因此不会触发 freelist 自环。
victim slab 的 `inuse` 最终变成 0，页面再次进入 buddy；victim 消息仍
保存着这 16 个 segment 指针，形成页级 UAF。

### 第二次重占并找到重复消息

再喷射 480 个带标记的 evil 大消息：

```c
buf[520] = 0x4556494c5f534547; /* "EVIL_SEG" */
buf[521] = evil_index;
```

使用 `MSG_COPY | IPC_NOWAIT | MSG_NOERROR` 无损查看每个 victim 消息。
若其 segment 内容已经变成 evil 标记，就说明 victim 与 evil 两条消息
引用同一个 2 KiB 对象。

一次成功的本地运行中找到：

```text
victim=9, evil=42
```

具体下标受页分配随机化影响，利用只依赖标记搜索，不硬编码结果。

### 从重复消息转成重复 pipe_buffer

找到重复消息后按以下顺序操作：

1. 删除 victim 消息，释放共享 segment；
2. 把空管道 A 扩容到 32 页，分配 `32 * sizeof(pipe_buffer) = 1280`
   字节的 ring array，进入 `kmalloc-cg-2k` 并重占 segment；
3. 删除 evil 消息，再次释放同一对象；
4. 把空管道 B 扩容，A/B 的 `pipe_buffer` 数组开始重叠；
5. 关闭空管道 A，立即用管道 C 重占；
6. 对 C 执行 `write -> read -> write`，把活动 buffer 移到 ring 下标 1；
7. 关闭仍为空的 B，把数组再次释放；
8. 发送全零 attack 大消息，用可控的 `msg_msgseg` 重占数组；
9. 再向 C 写入一次。下标 1 已被清零，内核会在下标 2 新建有效
   `pipe_buffer`；
10. 读取 attack 消息，泄露下标 2 的 `page`、`ops`、`flags` 和
    `private`。

消息文本中，segment 数据从 qword 507 开始，但跳过了对象开头的
`msg_msgseg::next`。因此：

```text
pipe_buffer[1] -> msg buffer qword 511
pipe_buffer[2] -> msg buffer qword 516
```

### 构造物理页任意读写

任意读使用 ring 下标 1：

1. 删除 attack 消息，取回当前 pipe array 内容；
2. 在 qword 511 伪造 `pipe_buffer[1]`；
3. 把 `page` 改成目标 `struct page *`；
4. 设置 `offset=0, len=0xff8`，恢复合法的 `ops/flags/private`；
5. 重新发送 attack 消息，覆盖管道 C 的数组；
6. 从 C 读取 `0xff0` 字节。

读取后还剩 8 字节，管道的 tail 不会前移，可以反复改写同一个
`pipe_buffer[1]`。

任意写使用最后一个活动 buffer，即 qword 516 的
`pipe_buffer[2]`：

1. 设置目标 `page`；
2. 设置 `offset=0, len=0`；
3. 恢复带有 merge 标志的 `flags` 和合法 `ops`；
4. 向管道 C 写入 `0xff0` 字节。

`pipe_write()` 会把数据 merge 到伪造的目标页中。

### 定位当前进程的 cred

泄露出的 `pipe_buffer::page` 位于 vmemmap。目标只有 128 MiB 内存，
`struct page` 大小为 `0x40`，因此物理页数组的有效偏移远小于
256 MiB。可以直接按 256 MiB 粒度取整：

```c
vmemmap_base = leaked_page & ~0xfffffffULL;
```

`pipe_buffer::ops` 指向 `anon_pipe_buf_ops`。附件 vmlinux 中其静态地址
为：

```text
anon_pipe_buf_ops = 0xffffffff832624c8
```

所以也可以直接计算 KASLR slide：

```c
slide = leaked_ops - 0xffffffff832624c8;
kernel_text = 0xffffffff81000000 + slide;
```

提权不需要执行内核 ROP。先把当前进程名设置为唯一字符串：

```c
prctl(PR_SET_NAME, "d3kheap2pwn");
```

随后按：

```c
vmemmap_base + pfn * 0x40
```

扫描物理页并搜索该字符串。Linux 7.1 的 `task_struct` 中仍保持以下相邻
关系：

```text
real_cred
cred
cached_requested_key
comm[16]
```

所以命中 `comm` 后：

```c
real_cred = ((uint64_t *)comm)[-3];
cred      = ((uint64_t *)comm)[-2];
```

要求两者相等且是合法的 direct-map 地址。把 cred 地址按 256 MiB 向下
取整即可得到候选 `page_offset_base`，再通过原始 uid/gid 验证对应物理页。

最后把 cred 前缀中的：

```text
uid gid suid sgid euid egid fsuid fsgid
```

全部改成 0，并用物理页任意写写回。此后当前进程可以直接打开
`/flag`。

### KFENCE 处理

本地直接运行旧版 exploit 时，KFENCE 曾采样到一个自定义 cache 对象；
第二次释放会触发 `KFENCE: invalid free`。

最终利用没有修改远端 sysfs，也不依赖 `skip_kfence=1`。由于 KFENCE
每 100 ms 只采样下一次 slab 分配，在两段敏感的 challenge ioctl 分配
之前执行：

```text
等待 120 ms
分配并立即删除一个很小的 msg_msg
立即完成一整段 challenge 对象分配
```

若采样门已经打开，它会被无害的 msg cache 分配消耗；随后的 36 次或
264 次 ioctl 分配在新的 100 ms 窗口内完成。最终版本已在未设置
`skip_kfence` 的原始本地配置中复跑成功。

### 官方记录的两条非预期解

官方 PDF 说明，普通版还存在两条不需要完成 sheaf 利用链的非预期路径，revenge 版本对它们都做了修复。

第一条来自 BusyBox `/etc/inittab` 的错误配置：

```text
::askfirst:/bin/ash
```

攻击者可以制造大量内存压力触发 OOM Killer，使阻塞启动流程的低权限进程被杀死，并让 BusyBox init 继续进入 `askfirst` 动作。由于该动作直接以 root 启动 `/bin/ash`，控制台会出现 root shell。这个技巧的关键不是模块内存破坏，而是把资源耗尽与 init 的高权限重启语义组合起来。官方引用的 [CTF Wiki OOM shell 条目](https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/tricks/out-of-mem-shell/)进一步记录了这类错误配置；理解本题只需记住，低权限用户可控的 OOM 事件不应把启动流程导向无认证的 root shell。

第二条与 `d3kbus` 普通版相同，是宿主 QEMU TCG 的 `access_ptr()` 无符号减法下溢。页尾 x87 80 位跨页访问可以获得越界 host 指针，配合页表喷射形成任意物理内存读写，再修改内核 `setuid` 检查提权。它绕过了 `d3kheap2pro.ko`，属于平台层非预期；具体边界错误与修复方式已经在 `d3kbus` WP 中展开。

这两条路径不能与本节前面的预期解混为一谈。预期解依赖模块 double free、Linux 7.1 sheaf 延迟批量释放和跨 cache 重占；OOM shell 依赖 init 配置，QEMU 链依赖虚拟化平台。区分漏洞所在层次，才能正确理解 revenge 到底修了什么。

### 脚本使用

在 WSL 中编译静态 exploit：

```bash
cd "/mnt/d/文档/新建文件夹/D3CTF2026/d3kheap2pro"
sh ./build.sh
```

远端上传器使用 pwntools，需要显式进入 `ctf-tools`：

```bash
source /home/kali/miniforge3/etc/profile.d/conda.sh
conda activate ctf-tools
python solve.py <新的靶机主机名>
conda deactivate
```

原版目录中的 `solve.py` 可直接作为 revenge 的上传器；它会：

1. 等待 guest shell；
2. 关闭 TTY echo；
3. gzip 压缩并分块 Base64 上传静态 ELF；
4. 比较本地和远端 SHA-256；
5. 运行 exploit 并提取输出中的 flag。

脚本不硬编码临时靶机地址，实例更新后只需替换命令行参数。

### 原版利用的本地验证记录

未关闭 KFENCE 的本地成功输出节选如下：

```text
[+] custom order-3 slab released (used 300 challenge slots)
[+] victim msg_msgseg spray reclaimed the released page
[+] victim kmalloc-cg-2k slab released through stale sheaf batches
[+] evil msg_msgseg spray reclaimed the dangling victim page
[+] overlapping messages: victim=9, evil=42
[+] pipe_buffer.page = 0xfffffcb7401ebc40
[+] pipe_buffer.ops  = 0xffffffffb78624c8
[+] vmemmap_base     = 0xfffffcb740000000
[+] kernel text      = 0xffffffffb5600000 (slide 0x34600000)
[+] task pfn=0x150, cred=0xffff88f605ec0840
[+] page_offset_base=0xffff88f600000000
[+] uid=0 euid=0 gid=0 egid=0
[+] flag: flag{arttnba3_t3s7_f1@9!}
```

## 方法总结

这道题真正的难点不是模块 double free 本身，而是 Linux 7.1 对旧利用链
的两处改变：

1. 公开的 `kmem_cache_free()` 会拒绝跨 cache 释放；
2. 普通 SLUB 快速路径改成了 per-CPU sheaf，释放与修改
   `slab->inuse` 之间出现了可控延迟。

最终利用把这个延迟变成了新的跨 cache 释放路径：

```text
refcount 错误
    -> 不同地址组成的 stale sheaf
    -> 填满 barn，延迟 bulk free
    -> 自定义 order-3 slab 归还 buddy
    -> msg_msgseg 重占
    -> stale sheaf 内部 bulk free 绕过公开 cache 校验
    -> victim/evil 消息双引用
    -> pipe_buffer 数组双引用
    -> 物理页任意读写
    -> 扫描 task_struct 并清零 cred
```

几个决定稳定性的细节是：

- 不能在同一次 bulk free 中放入重复地址，否则 hardened freelist 会形成
  自环并触发 `BUG()`；
- 必须按 12-object sheaf 和 16-object slab 的真实 LIFO refill 顺序选择
  slot；
- 必须保留目标页的 buddy slab，防止目标块合并成 order-4；
- 喷射数同时整除 12 和 16，并覆盖 page allocator shuffle；
- KFENCE 样本要由无害 cache 提前消费；
- 任意读使用 pipe ring 下标 1，任意写使用最后一个可 merge 的下标 2。

这些约束共同构成了 `d3kheap2pro` 预期解相比旧版题目的主要修复与绕过点，在 revenge 中仍然成立。

赛后材料披露的 OOM root shell 与 QEMU TCG 越界均是旁路非预期：它们分别利用启动配置和宿主模拟器，完全跳过上述 sheaf 链。revenge 把 `askfirst` 改为 `poweroff`，并在宿主 TCG 中补上无符号范围检查；这些修复不会改变 `d3kheap2pro.ko` 的 double free。由此，revenge 的正确路线仍是 sheaf 延迟 bulk free、跨 cache `msg_msgseg` 重占、重叠 `pipe_buffer` 和物理页读写，而不是继续寻找已封堵的旁路。
