# TSGCTF2020 RACHELL WP

## 题目简述

题目实现了一个没有 `cat` 的内存文件系统，用户只能用 `mkdir`、`touch`、`echo`、`rm`、`cd`、`ls` 和 `pwd` 操作节点。预期解刻意避免依赖地址泄漏，要求利用文件缓冲区的 UAF/双重释放完成 House of Corrosion，在只知道 libc 内部低位偏移的条件下篡改 `stderr` 并取得代码执行。

漏洞位于 `sub_rm`。从当前目录直接删除文件时，节点会从父目录解除链接；但使用跨一层路径删除文件时，代码只释放 `target->buf`，既不清空指针，也不解除节点：

```c
if (target->p != cwd && target->type == FIL) {
    if (target->buf != NULL)
        free(target->buf);
}
```

同一个文件随后仍可被 `echo` 写入或再次 `rm`，形成 UAF 和 double free。另一个重要细节是 `write2file` 先按输入长度申请内存，却把 `\r` 当作内容终止符；发送一长串 `\r` 可以申请任意大块但实际复制 0 字节，从而保留块内原有指针和未覆盖数据。

## 解题过程

先建立 `test1` 至 `test6` 六个目录，并用大量 `touch` 预先分配节点结构。这样后续 `echo` 主要改变内容缓冲区，节点分配带来的堆布局扰动较小。通过从父目录执行 `rm("./testN/file")`，保留指向已释放 buffer 的文件节点；重新 `echo` 时即可在旧地址写入数据。

第一阶段在 `test1` 中布置两个约 `0x450` 的大块及垫块。释放后利用 UAF 改写 chunk size，把一个块接入 largebin，并设置 `NON_MAIN_ARENA` 位，为最后触发 allocator 错误准备条件。同时对 unsorted-bin 块的 `bk` 做低 2 字节覆盖，使链表写落到 `global_max_fast - 0x10`：

```python
gmf = 0xc940
rm("test1/a")
cd("test1")
echo("a", p64(0) + p16(gmf - 0x10))
echo("hoge", b"G" * 0x450)
```

libc 基址随机化不会改变页内低 12 位，但目标需要猜额外半个字节，因此官方脚本注明这一阶段约有 4 bit 不确定性，失败时重新连接。成功后 `global_max_fast` 被扩大，原本远超正常 fastbin 范围的尺寸也会按 fastbin 索引处理；越界索引由此把释放链表写入 libc 的全局对象区域。

第二阶段在 `test2`、`test3` 中各构造一组重叠块。脚本只改相邻 chunk 指针的最低字节，让重新分配的块覆盖原对象，并伪造足够多的 `0x31` 小块头。对目标字段的距离使用：

```python
def formula(delta):
    return delta * 2 + 0x20
```

这是因为扩大的 `global_max_fast` 让某个 libc 地址差值映射到对应 fastbin 索引。配合 UAF 释放、重新申请及部分指针覆盖，可以完成 House of Corrosion 的两类“移植”：把 libc 中已有的指针值搬到 `stderr` 指定字段，或只改目标指针的低位而保留未知高位。

官方脚本依次调整以下关键状态：

- 将 `stderr->_mode` 和 `stdout->_mode` 置为 1，清理 `stderr->_flags`，并把 `stderr->_IO_write_ptr` 设为极大值；
- 把 `__morecore` 中的有效 libc 指针移植到 `stderr->_IO_buf_end`，避免需要知道完整地址；
- 将 `stderr` 的 vtable 低位改为 `_IO_str_jumps - 0x20`，使错误输出进入 `_IO_str_overflow` 路径；
- 把 `__morecore` 的值移植到 `_s._allocate_buffer`，再部分覆盖为 libc 内的 `call rax` gadget；
- 在 `stderr->_IO_buf_base` 中布置相对 one-gadget 的偏移，使 `_IO_str_overflow` 的长度计算最终把目标地址送入 `rax`。

最后再执行一次 `echo`，促使 allocator 处理预先设置了 `NON_MAIN_ARENA` 的异常 largebin chunk。`malloc_printerr` 尝试向已经伪造的 `stderr` 报错，伪造 vtable 将控制流引入 `_IO_str_overflow`，再经 `call rax` 到 one-gadget，得到 shell。仓库中的 flag 为：

```text
TSGCTF{beer_is_delicious_if_you_dont_taste_it_6592867821310}
```

源码还存在一个非预期泄漏：`sub_pwd` 只检查根目录下一层的名称，深层目录名会未经 `ascii_check` 直接输出。利用堆重叠把指针写进深层目录名后，可以泄漏 libc，再走普通 hook 覆盖路线；这比无泄漏的预期解短，但不体现题目设计的 House of Corrosion 主线。

## 方法总结

本题把文件系统接口转化为堆操作原语：跨目录删除产生 UAF/double free，`\r` 的错误终止语义实现“只申请、不覆盖”，而 `touch` 将节点和内容块的分配解耦。预期解再通过 unsorted-bin attack 扩大 `global_max_fast`，用超范围 fastbin 索引和局部指针改写逐字段伪造 `stderr`，最终借 allocator 报错触发 FSOP。复现这类无泄漏利用时，必须区分固定的页内偏移、需要爆破的 ASLR 位和由现存指针完成的地址移植，不能把脚本中的常量误写成完整已知地址。
