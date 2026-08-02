# TJCTF2022 babyheapng

## 题目简述

题目复用了 babyheap 的菜单与“释放后不清空槽指针”漏洞，但运行库换成采用 mallocng 分配器的 musl 1.2。旧版空闲块链伪造不再适用：mallocng 用 `meta_area`、`meta`、`group` 管理 slot，并以 `__malloc_context` 中的随机 secret 校验元数据。程序仍有 seccomp 限制，最终需要构造 ORW 链读取 `flag.txt`。

## 解题过程

先制造 UAF，把新分配的数据解释成旧 `Slot` 的 `size`，令 `view(2)` 输出 100 字节。取泄漏末尾的地址减 `0x97ce0`，可恢复 musl 基址：

```python
malloc(0, 16, b'a')
malloc(1, 64, b'a')
for i in range(5):
    malloc(i + 2, 16, b'a')
free(0)
free(2)
malloc(7, 16, p64(100))
view(2)
libc.address = u64(io.recv(48)[-8:]) - 0x97ce0
```

接着再次利用 UAF，把旧槽的缓冲区指针改为 `__malloc_context`，从而泄漏校验 secret。官方解以 `libc.address - 0x22000` 作为伪元数据页，在页边界附近依次布置合法 secret、`meta_area`、`meta`、`group` 和 slot 描述。释放由这些结构描述的伪 slot 后，它会被 mallocng 接受并进入可分配状态。

第二次改写伪 `meta.mem`，使后续分配返回到 `__stdin_FILE - 0x30`。在 FILE 区域布置 `flag.txt`、栈迁移 gadget 和系统调用 ROP 链，依次执行 `open`、`read`、`write`。退出菜单触发清理路径后，服务输出：

```text
tjctf{mallocng_1s_n0thing_but_p4in_d722cda0a97e60f6}
```

## 方法总结

本题展示了同一上层 UAF 在不同分配器版本中的完全不同利用方式。mallocng 的重点不是传统 fd/bk 指针，而是伪造一组内部一致、页位置正确且带有效 secret 的对象图。可靠分析应先恢复库版本和管理结构，再设计任意写；版本差异不是偏移微调，而是利用模型本身发生了变化。
