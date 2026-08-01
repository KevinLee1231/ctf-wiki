# GlacierCTF2023 - flipper

## 题目简述

题目基于 SWEB 教学操作系统，新增 syscall 69：用户可以指定任意地址和位号，让内核执行 `*address ^= 1 << bitnum`。题面称有三次 bit flip，作者简述又称一次，而仓库实现用 `flipped_already != 0` 实际限制为一次；官方 solver 因此先修改限制逻辑，再获得无限次翻转。

## 解题过程

第一次调用针对内核地址 `0xffffffff8010a604` 的 bit 0。该处原字节为 `0x74`，翻转后成为 `0x75`，把检查 `flipped_already` 的条件跳转反向，使后续调用不再返回 `-2`：

```c
flipBit((void *)0xffffffff8010a604, 0);
```

接着修改 `Syscall::write` 中拒绝内核地址缓冲区的分支。目标五字节为 `e9 9c 00 00 00`；按官方脚本逐位异或后，每个字节都变成 `0x90`，即五个 NOP：

```text
e9 XOR 79 = 90
9c XOR 0c = 90
00 XOR 90 = 90
00 XOR 90 = 90
00 XOR 90 = 90
```

范围检查被抹掉后，用户态可把内核地址直接作为 `write(1, buffer, size)` 的源缓冲区。官方构建中 flag 位于 `0xffffffff8013294d`，长度为 27：

```c
write(1, (void *)0xffffffff8013294dULL, 27);
```

输出为：

```text
gctf{1ll_g3t_3CC_n3xt_y34r}
```

作者短 WP 还给出预期思路：定位 identity mapping 的 PML4、PDPT、PD 条目，翻转各级 `user_access` 位后从用户态扫描物理内存。该路线依赖题面宣称的多次翻转；上文则忠实描述仓库官方 solver 针对实际“一次限制”的自修改内核路径。

## 方法总结

任意内核地址 bit flip 已是强写原语，最有价值的首个目标通常是原语自身的次数或权限检查。文档、题面与实现发生漂移时，应以实际代码和官方可运行 solver 为主，并明确记录另一条设计意图，不能把两个版本拼成一条不存在的利用链。
