# mybookv2

## 题目简述

第二版为作者增加了 16 位引用计数，但沿用了有缺陷的 `read_strline`：当读取长度为 0 时，`read` 返回 0，随后仍执行 `buf[n-1] = 0`。对作者名编辑时，这个负一字节恰好覆盖引用计数的高字节。

## 解题过程

创建作者后关联 256 本书，使引用计数从 1 变为 `0x0101`。随后以长度 0 编辑作者名；`name[-1]` 是 `refcnt` 的高字节，因此计数被改成 `0x0001`。删除任意一本关联图书会把计数减为 0 并释放作者，但其余 255 本书仍保存该作者指针，形成稳定 UAF。

利用悬空 `Author` 指针与可控描述块重叠，先泄露 heap，再借大块释放泄露 libc。最后伪造 tcache 块，把下一次分配导向 `__free_hook`，写入 `system` 后释放 `/bin/sh`：

```python
# 关键触发顺序
for i in range(256):
    create_book(i, 0, b"A")
edit_author(0, length=0, data=b"")
delete_book(0)  # refcnt: 1 -> 0，作者被提前释放
```

最终得到：

```text
n00bz{I_thought_reference_counter_suppossed_to_be_safe_again}
```

## 方法总结

引用计数只能在所有写入和增减操作都正确时提供安全性。本题并非 16 位计数自然溢出，而是零长度读取造成的单字节向前写，把 `0x0101` 精确降为 `0x0001`；这一点决定了为何需要先创建 256 个引用。
