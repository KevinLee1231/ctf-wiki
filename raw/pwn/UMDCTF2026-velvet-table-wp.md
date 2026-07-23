# velvet-table

## 题目简述

题目是一个带索引混淆的堆菜单程序。连续释放同尺寸块时，前 7 次会把槽位指针清空；当对应 tcache 计数已满后，程序仍执行 `free`，却保留槽位指针，产生可读写的悬空指针。栈上还刻意放置了一个大小为 `0x111` 的伪 smallbin 块头，以及最终的 `payout_fn` 和完整性标签。

仓库没有提供官方 solve，因而本题只能依据源码给出可验证的利用链和关键约束，不虚构特定 glibc 地址下的最终字节序列。

## 解题过程

程序首先打印：

```text
table marker = table_key ^ 0xC0DEC0DE
```

因此可直接恢复 `table_key`，并进一步计算：

```text
map_key = (table_key ^ 0xA7) & 0x0f
real_slot = ((user_slot ^ map_key) + 3) & 15
```

`inspect` 输出的每个字节又只与公开算法 `leak_mask(table_key, slot, i)` 异或，所以能够还原真实堆数据。利用主线为：

1. 反算用户座位到真实槽位的映射；
2. 对目标尺寸填满 7 个 tcache 项；
3. 再释放同尺寸块，使其进入 unsorted/smallbin，而槽位保留悬空指针；
4. 用 `inspect` 读取释放块元数据，获得堆或 libc 相关指针；
5. 让 `ledger >= 7` 后调用 `settle-ledger`，启用不再隔字节写入的 `memcpy` 模式；
6. 通过 UAF 与最长 `0x400` 字节的越界写修改 smallbin 链。

栈上的伪块头地址由

$$
\text{table\_key}=(\text{stack}\gg4)\oplus0x5A17C3D9
$$

泄露了足够多的栈位置关系；`dealer-note` 又允许在满足 `ledger >= 5` 且操作计数为 4 的倍数时，经可逆的 `note_mask` 精确写入伪块头前 32 字节。由此可以构造 smallbin 链，使后续分配落到或重叠 `local`。

取得对 `local` 的写能力后，把：

```text
payout_fn  = win
payout_tag = (table_key << 32) ^ win ^ 0x686f7573655f6564
```

同时写入。最后调整 `ledger` 为奇数并选择 `payout`，完整性检查通过后调用 `win`：

```text
UMDCTF{smallbins_still_love_the_stack_when_the_house_sets_the_table}
```

## 方法总结

这道题把关键条件都写进了源码：tcache 满后残留的悬空指针、可逆掩码、栈地址派生的 marker、栈上伪 smallbin 块以及带标签的函数指针。正确分析顺序是先恢复索引和掩码，再建立 UAF 泄露/写原语，最后按实际 glibc 版本完成 smallbin 到栈的重叠；仅覆盖函数指针而不同时更新标签不会成功。
