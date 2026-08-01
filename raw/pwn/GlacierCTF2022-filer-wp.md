# GlacierCTF2022 - Filer

## 题目简述

程序维护 `storage[16]` 指针数组和 `sizes[16]` 长度数组，提供新增、修改和删除功能。它会把同一个 logo 文件反复输出，但没有 UAF；真正入口是索引边界的 off-by-one，随后需要把越界长度升级为堆溢出、libc 泄漏和 tcache poisoning。

## 解题过程

`add` 与 `change` 都只拒绝 `index > 0x10`，所以索引 16 被错误放行。两个全局数组相邻，`storage[16]` 恰好越过指针数组边界并覆盖 `sizes[0]`、`sizes[1]`。在索引 16 新增条目后，写入的堆指针低 32 位成为 `sizes[0]`，使 `change(0)` 可以向原本约 `0x70` 大小的 chunk 写入极长数据。

先分配四个 100 字节条目并依次释放 3、2、1，建立目标尺寸的 tcache 链。利用放大的索引 0 写入覆盖相邻空闲 chunk 的 size 与 next 指针：

```python
payload = b"X" * 0x68
payload += p64(0x71)
payload += b"\x90\x32"       # 部分覆盖 tcache next
change(0, payload)
```

官方环境没有 safe-linking。两字节部分覆盖把下一次同尺寸分配引向 libc 的 `_IO_2_1_stdout_` 附近；ASLR 的未覆盖低熵位不匹配时进程会失败，所以官方脚本用重连循环重试。取得重叠分配后写入伪造的 FILE 头：

```python
fake = b"B" * 8 + p64(0x1e1)
fake += p64(0xfbad2488) + b"\x70"
```

`0xfbad2488` 触发 stdout 输出内部缓冲区，从返回的 6 字节指针计算 libc 基址。接着释放一个可控 chunk，再用同一堆溢出把 tcache next 改为 `__free_hook`；连续两次分配后将 `system` 写入 hook。最后分配保存 `/bin/sh\0` 的条目并删除它，`free(ptr)` 就变为 `system(ptr)`。在 shell 中读取：

```text
glacierctf{Now_1Mag1n3_L4t3st_L1bc?!}
```

## 方法总结

边界检查应使用 `index >= count`。本题的 off-by-one 没有直接访问一个普通条目，而是跨数组把堆地址写进长度元数据，间接制造大范围 heap overflow；之后通过 tcache 部分指针覆盖重叠 stdout 泄漏 libc，再以 `__free_hook` 完成控制流劫持。
