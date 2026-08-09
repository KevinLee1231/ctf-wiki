# heapMage

## 题目简述

程序提供固定索引的堆块分配、释放和编辑。小块实际用户区有 `0xc0` 或 `0xf0` 字节，但编辑路径统一读取最多 `0xf0` 字节，因此编辑 `0xc0` 块时可以向后溢出 `0x30` 字节，覆盖下一块的 `prev_size`、`size` 和链表指针。程序自定义的 `faded()` 只检查部分相邻 size/bk 是否落在 heap，挡不住经过布局的伪块。

附件使用较新的 glibc，传统 hook 已不可用。官方利用以 House of Rust 取得 tcache metadata 重叠，再制造 libc 泄漏与任意 `malloc`，最后用 House of Apple 2 劫持 `_IO_2_1_stderr_`。

## 解题过程

### 1. 用溢出伪造两个 largebin 块

先按官方 `exp.py` 的固定分配顺序布置小块和两个大块。利用前置 `0xc0` 块的越界编辑，把后继区域分别伪造成约 `0x4e0` 与 `0x4d0` 的 chunk，并补好后续 chunk 的 `prev_size`/`PREV_INUSE`，使释放和整理时能通过 glibc 一致性检查。

将较大的伪块先送入 largebin，再把较小块送入 unsorted。修改前者的 `bk_nextsize` 低 16 位，使后者作为新的最小 largebin chunk 插入时，把指针写到 `tcache_perthread_struct` 附近。利用只改低位的方式需要枚举少量 heap ASLR 低位；失败实例直接重连，不应把错误布局继续带到后半段。

### 2. tcache stashing 取得 metadata 重叠

提前准备一组 `0xd0` smallbin/tcache 块，调整链表顺序，让一次 tcache stashing unlink 沿着真实小块、伪 metadata 块和第二个 largebin 块移动。完成后，一次对应大小的 `malloc` 会返回 `heap+0x90`，也就是 tcache entries 区域。

此时编辑该“普通块”实际上是在改 `tcache_perthread_struct`，可以控制各 size class 的单链表头。官方脚本刻意在破坏 largebin 前准备好后续要用的 `0x100` tcache，避免 glibc 2.39 再次扫描已损坏的 largebin。

### 3. 泄漏 libc 并形成任意分配

在 metadata 重叠中伪造一个 `0x100` chunk，使其进入 unsorted/smallbin；排序时写入的 libc 指针覆盖 `entries[14]`。随后只改该 entry 的低位，让 `malloc(0xf0)` 返回 `_IO_2_1_stdout_`，再写入经典的 `_flags=0xfbad1800` 和收缩 `_IO_write_base`，从程序输出中泄漏 stdout 附近地址，计算 libc 基址。

泄漏后 tcache count 仍为正。直接重写 `entries[14]` 为任意对齐地址，下一次 `malloc(0xf0)` 就会把该地址当作用户区返回，形成可重复的任意地址分配/写入原语。

### 4. House of Apple 2 控制流劫持

在 libc 可写区分别放置伪 `_IO_wide_data`、伪 wide vtable 和可用的 lock；把 wide vtable 的 `__doallocate` 槽写成 `system`。最后把 `_IO_2_1_stderr_` 改造成：

```text
_wide_data -> fake_wide_data
vtable     -> _IO_wfile_jumps
_mode      -> 1
command    -> "  sh -i"
```

选择退出菜单触发 stdio 清理路径，经过 `_IO_wfile_jumps` 调到伪 `__doallocate`，等价于 `system(command)`。取得 shell 后读取 flag。官方 `writeup/writeup.md` 中的 `exp.py` 已包含每个 heap 偏移、低位枚举和分阶段调试开关。

## 方法总结

本题的起点只是 0x30 字节线性溢出，但真正难点是保持现代 glibc 各 bin 的一致性。House of Rust 的目标不是直接覆盖函数指针，而是先把 tcache metadata 变成可分配对象；有了它，libc 泄漏和任意地址 `malloc` 才能稳定衔接。复现时应分四个检查点：largebin 插入成功、metadata chunk 返回、stdout 泄漏基址正确、任意分配命中目标。若跳过这些检查，低位猜测失败往往会被误判为 Apple 2 结构错误。
