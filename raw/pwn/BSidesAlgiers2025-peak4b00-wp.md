# BSidesAlgiers2025 - Peak4b00

## 题目简述

题目是 glibc 堆利用。程序维护 10 个 `slots`，支持申请、释放、编辑和 `peakaboo` 查看。`free_slot()` 释放后不清空指针也不重置大小，因此同时留下 UAF 读、UAF 写和 double-free 条件；`peakaboo()` 又允许未经边界检查的有符号偏移读取。官方 solver 将这些原语组合为：泄露 safe-linking 堆页、tcache poisoning 覆盖 banner、越界泄露 libc、再次 tcache poisoning 改写 `_IO_list_all`，最后用 FSOP 在退出刷新阶段调用 `system("sh")`。

## 解题过程

### UAF 与第一次堆泄露

释放函数的缺陷很直接：

```c
free(slots[idx]);
puts("Slot freed.");
```

`slots[idx]` 仍指向原 chunk，所以 `edit_slot()` 和 `peakaboo()` 都能继续操作该 chunk。先申请两个 `0x50` chunk，再依次释放 0、1。tcache 中末端 chunk 的 `next=NULL` 会按 safe-linking 编码为当前 chunk 地址右移 12 位，因此读取 slot 0 开头即可得到堆页值 $H=heap\_addr\gg12$，堆基址为 $H\ll12$。

### 覆盖 banner，重置一次性读取门

`peakaboo()` 每次结束会在 banner 的字符串终止符后写入 `Z`，下次调用看到该字节非零就退出：

```c
if ((char)banner[strlen(banner) + 1]) {
    exit(1);
}

/* 完成一次读取后设置下一次检查会命中的哨兵。 */
*(banner + strlen(banner) + 1) = 'Z';
```

官方脚本没有放弃第二次读取，而是利用已释放 slot 1 的 UAF 写改掉 tcache `fd`。safe-linking 下应写入：

$encoded\_fd = target \oplus (chunk\_address \gg 12)$

目标取 `heap_base + 0x480`，即 banner 数据区。连续申请两次后，第二个返回值落到 banner；写入一份等长菜单文本并在末尾放两个 `\0`，便把第一次 `peakaboo` 写入的哨兵重新清零。

### 越界泄露 libc

`peakaboo` 的第二个索引没有范围检查：

```c
printf("Here you go: %s", slots[index] + idx * 8);
```

使用 slot 0 和 `idx=-43`，即向前读 $43\times8=344$ 字节，可泄露到 libc 指针。官方 solver 针对随题提供的 libc 使用偏移 `0x202030` 计算基址。这个偏移是版本相关值，远程复现必须显式加载附件中的 `libc.so.6`，不能默认沿用宿主 libc。

### `_IO_list_all` 劫持与退出触发

接着申请四个 `0x100` chunk，在 slot 7 所在的 `heap_base+0x8c0` 构造 fake `_IO_FILE`。关键字段包括：

```python
fake_file = flat({
    0x00: b"  sh",
    0x28: libc.sym["system"],
    0x88: libc.address + 0x205710,
    0xa0: fake_addr + 0xd0 - 0xe0,
    0xd0: fake_addr + 0x28 - 0x68,
    0xd8: libc.sym["_IO_wfile_jumps"],
    0xe8: fake_addr + 0x28 - 0x88,
}, filler=b"\0")
```

释放 slot 4、5 后，再通过 slot 5 的 UAF 写把 tcache 链毒化到 `_IO_list_all`；两次申请后，slot 9 即指向 `_IO_list_all`，写入 `fake_addr`。此时再次选择菜单项 4，banner 哨兵触发 `exit(1)`。glibc 在退出时遍历并刷新 `_IO_list_all`，伪造的 wide-file 调度链最终以 fake FILE 起始处的 `"  sh"` 为参数调用 `system`，得到 shell。

仓库内 `solution/solve.py` 已实现上述完整顺序；附件 flag 为：

`shellmates{p3akab00!!_F$op_Exit_3xploitation_i$_cr444444zy}`

本轮只完成源码、官方 solver 和随题 libc 偏移的静态对账，没有启动容器执行交互利用。

## 方法总结

- 释放后不清空全局指针会把 UAF 读写、重复释放和 tcache poisoning 同时暴露出来。
- safe-linking 不是随机加密；获得任一同页 chunk 的地址高位后，就能按 $target\oplus(addr\gg12)$ 构造合法链指针。
- 一次性漏洞检查本身也可能存放在可覆盖堆对象中。本题先用 tcache poisoning 重置 banner 哨兵，才获得第二次越界读取。
- 新版 glibc 缺少传统 malloc/free hooks 时，可把任意写转向 `_IO_list_all`，再用进程退出的标准 I/O 刷新路径触发 FSOP。
