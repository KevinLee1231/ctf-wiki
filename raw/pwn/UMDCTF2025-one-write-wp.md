# UMDCTF 2025 - one-write

## 题目简述

程序启动时分配并立刻释放一个 `0x5f8` 字节 chunk，但把指针继续保存在全局变量
`the_chunk`：

```c
the_chunk = malloc(0x5f8);
free(the_chunk);
```

菜单中的普通分配没有写入功能，而 `Write` 和 `Show` 始终通过这个悬空指针读写
`0x5f8` 字节。攻击者因此可以查看和改写后来复用该区域的堆块及 allocator 元数据。
目标是逐步泄漏 libc、heap、stack 和 PIE，再借 smallbin/tcache stashing 把
`the_chunk` 本身改成栈地址。

## 解题过程

### 1. 从同一悬空指针取得四类地址

先申请两个 `0x418` 用户区的 chunk 再释放。它们进入 unsorted bin 后，空闲块头部
的 `fd`/`bk` 指向 `main_arena`；`Show` 通过 `the_chunk` 读出 `fd`，减去已知偏移
即可得到 libc 基址。

接着申请并释放 `0xf8` 用户区的 tcache chunk。tcache safe-linking 把 `fd`
保存为：

```text
encoded_next = next ^ (chunk_address >> 12)
```

当链尾的 `next` 为零时，读出的值就是 `chunk_address >> 12`，左移 12 位恢复 heap
基址。

有了 libc 和 heap 后，向 tcache `fd` 写入编码后的 `__libc_argv`：

```python
write(p64(libc.sym["__libc_argv"] ^ (heap >> 12)))
```

两次分配使 tcache 返回该伪造目标。再释放一个相关 chunk 并读取数据，需要同时撤销
真实堆块与伪造目标地址造成的两层 safe-linking 掩码：

```python
stack = leak ^ (heap >> 12) ^ (__libc_argv >> 12)
```

同样的方法把下一次伪造目标指向栈上的 `_start` 返回地址附近，读取代码指针并减去
`_start` 符号偏移，得到 PIE 基址。

### 2. 用 smallbin stashing 改写 `the_chunk`

最终目标不是直接把某个 tcache 项分配到栈，而是先修改全局悬空指针。官方利用反复
填满、清空对应 size class 的 tcache，并在 `the_chunk` 可覆盖的区域中搭建伪造
smallbin 双向链。

第一次 stashing 构造 8 个节点的指针链，并让尾节点的 `bk` 指向
`&the_chunk - 0x10`；后续再借 `main_arena + 336` 附近的 smallbin 头建立能通过
完整性检查的前后向关系。由于直接使用 `0x100` size class 会令
`the_chunk->fd == bin` 干扰检查，官方脚本最后切换到 `0xf0` size class。

最终伪造关系可概括为：

```text
smallbin head <-> fake chunk <-> (&the_chunk - 0x10) <-> smallbin head
```

同时在空 tcache 头中放入经过三项异或修正的目标栈地址：

```python
encoded = target_stack \
        ^ ((smallbin_addr + 0x20) >> 12) \
        ^ (the_chunk_addr >> 12)
```

最后一次 tcache stashing 写回全局区域，使 `the_chunk` 的值变成选定的栈返回地址。

### 3. 把固定写变成栈 ROP

此时菜单的 `Write` 已经不再写堆，而是直接覆盖 `write_chunk` 的返回链。已知 libc
基址后，写入：

```python
payload = flat(
    pop_rdi,
    next(libc.search(b"/bin/sh\0")),
    ret,
    libc.sym["system"],
)
write(payload)
```

`ret` 用于保持调用 `system` 时的栈对齐。函数返回后执行
`system("/bin/sh")`，读取：

```text
UMDCTF{but_look_at_this_its_j0hn_p0rk}
```

## 方法总结

本题把所有能力都限制在同一个悬空指针上，但这个指针覆盖范围足以观察并破坏 glibc
allocator 元数据。利用链的重点是：unsorted bin 泄漏 libc、safe-linked tcache
泄漏 heap、针对 `__libc_argv` 的 tcache poisoning 泄漏 stack、再泄漏 PIE，
最后通过 smallbin tcache stashing 写回全局指针。面对受限堆题时，不应只问
“能写几次”，还要问固定写入区域在不同分配阶段会与哪些元数据和对象重叠。
