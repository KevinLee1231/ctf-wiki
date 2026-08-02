# N1CTF 2021 - house_of_tataru

## 题目简述

题目使用 musl libc 的新 mallocng 分配器，并提供可影响申请尺寸、索引、编辑和显示的菜单。利用链不是传统 glibc house：先借 musl 对 BSS 空闲空间的复用与错误回显侧信道泄露 heap、PIE、libc，再伪造 `meta/group`，把分配返回位置导向任意地址，最后覆盖 `__funcs_on_exit` 的链表头并执行 ORW ROP。

官方 README 给出预期思路和利用脚本；[kileak 的参赛者复盘](https://kileak.github.io/ctf/2021/n1ctf21-tataru/) 对 musl 元数据检查与替代利用路径有更完整的调试记录。下面已将完成利用所需的关键机制写入正文。

## 解题过程

### 泄露 heap 与 PIE

musl mallocng 会把程序 BSS 中未使用的可写区域当作早期 allocator 存储，因此小块可能直接落入 PIE 映像附近；真正 heap 与 PIE 之间的页距离则由 ASLR 随机化。

官方脚本先布置多个 `0x10` 申请，触发一次超大失败申请，再用接近 chunk 边界的 `edit` 与 `show` 读出下一组 `meta` 指针：

```python
edit(0, b"a" * 287)
leak = show(0)
heap_base = u64(leak[:6].ljust(8, b"\x00")) - 144
```

仅有 heap 地址还不能直接推导 PIE。程序的 `edit` 在底层 `read` 失败时打印 `failed`，而不是立即终止。利用可控的 chunk offset 按页扫描候选距离：当偏移落到映射的 BSS/PIE 页面时，响应行为改变。官方脚本枚举 `0x100..0x2000` 个页，得到：

```python
pie_base = heap_base - page_offset * 0x1000
```

这比直接远程爆破 ASLR 稳定；README 记录的非预期解曾用约 7 小时远程撞中距离。

### 定位 libc 与伪造 mallocng 元数据

申请足够大的块会让 musl 通过 `mmap` 在 libc 附近建立映射，其 `meta->mem` 指向该区域。控制 chunk offset 后可越界读取对应 `meta`，由映射地址减固定偏移得到 libc 基址。

接下来需要伪造 mallocng 的 `meta`：

```python
class Meta:
    prev = next = group = 0
    avail_mask = free_mask = 0
    last_idx = 7
    freeable = 1
    sizeclass = 0
    maplen = 1
```

通常伪造 `meta->mem` 会在 `calloc` 的全零检查中失败。musl 逻辑为：

```c
void *p = malloc(n);
if (!p || (!__malloc_replaced && __malloc_allzerop(p)))
    return p;
n = mal0_clear(p, n);
return memset(p, 0, n);
```

官方解法不使用已被题目限制的 unsafe unlink，而是利用 malloc `enframe` 写入 chunk 索引的行为，把非零值写到 `__malloc_replaced`。这样 `calloc` 不再要求伪造分配天然为全零。随后把 `meta->mem` 指向目标地址，即可让下一次申请返回近似任意地址，形成任意写。

### 劫持退出回调并执行 ORW

musl 的退出处理维护如下链表：

```c
struct fl {
    struct fl *next;
    void (*f[32])(void *);
    void *a[32];
};
```

`__funcs_on_exit()` 从全局 `head` 开始，反向遍历函数槽并调用 `f[i](a[i])`。利用任意分配把 `head` 改为 BSS 中伪造的 `fl`，即可同时控制 RIP 与 RDI。

官方脚本使用 musl 中的栈迁移 gadget：

```text
mov rsp, qword ptr [rdi + 0x30]
jmp qword ptr [rdi + 0x38]
```

把 `a[i]` 指向伪造参数区，在 `+0x30` 放 ROP 栈地址、`+0x38` 放第一跳。题目有 seccomp，不能依赖 `system("/bin/sh")`，所以最终链执行：

```text
open("flag", 0)
read(3, buffer, 0x100)
write(1, buffer, 0x100)
```

选择退出菜单后触发伪造回调并打印 flag。仓库没有保存实际远端输出，因此不臆造具体 flag 字符串。

## 方法总结

本题的关键是理解 musl mallocng 的 `meta/group`，而不是套用 glibc tcache 模板。泄露阶段利用 allocator 对 BSS 的复用和错误分支侧信道；任意写阶段先处理 `calloc` 的零内存检查；控制流阶段则选择结构稳定的退出回调。存在 seccomp 时，应从一开始就按允许的 `open/read/write` 设计最终载荷。
