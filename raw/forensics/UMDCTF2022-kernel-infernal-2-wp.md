# UMDCTF2022 Kernel Infernal 2 Writeup

## 题目简述

本题沿用上一题的 Linux kdump，要求填写“第一个 CR3”的地址。对 x86-64 用户进程而言，`task_struct.mm` 指向 `mm_struct`，其中的 `pgd` 指向该地址空间的顶级页表。因此可以在 `crash` 中沿 `task_struct -> mm -> pgd` 读取目标值。

需要注意：硬件 CR3 保存的是页表根的物理地址及少量控制位，而 Linux 内核结构中的 `mm_struct.pgd` 是内核可访问的虚拟地址。题目给定的 flag 形式和官方结果实际要求后者。

## 解题过程

仍使用与转储匹配的调试内核：

```bash
crash vmlinux-5.4.0-99-generic \
  ubuntu20.04-5.4.0-99-generic-cloudimg-20220215.kdump
```

先用 `ps` 查看任务列表，并按题意选择第一个具有用户地址空间的任务。内核线程的 `task_struct.mm` 通常为 `NULL`，不能直接作为目标。取得任务结构地址后，分两步读取字段：

```text
crash> struct task_struct.mm <task_struct 地址>
crash> struct mm_struct.pgd <mm_struct 地址>
```

也可以在确认结构偏移后沿指针连续查看。目标 `pgd` 为：

```text
0xffff9b187a8e6000
```

最终 flag：

```text
UMDCTF{0xffff9b187a8e6000}
```

[参赛者复盘](https://sutharnisarg.medium.com/umdctf-2022-write-ups-c45cdef017bb)同样指出应从 `task_struct` 走到 `pgd`；仓库中的官方 flag 文件确认了最终地址。外链中的必要结论已写入正文。

## 方法总结

内存取证中必须区分硬件寄存器值、物理地址和内核虚拟指针。若题目真正要求可写回 CR3 的物理基址，还需把 `pgd` 虚拟地址转换为物理地址并处理低位标志；本题的判题值直接采用 `mm_struct.pgd`。遍历任务时还要跳过 `mm == NULL` 的内核线程。
