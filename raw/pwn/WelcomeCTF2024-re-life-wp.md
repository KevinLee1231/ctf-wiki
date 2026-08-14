# re:life

## 题目简述

题目组合了整数溢出、UAF、反复 `execve`、BSS 到堆溢出和 tcache 元数据控制。目标是枚举一次 BSS 与 heap 恰好相邻的 ASLR 布局，再覆写 `tcache_perthread_struct` 获得任意地址分配，最终把 ROP 链写到栈上执行 shell。

## 解题过程

### 枚举相邻布局

`time_skip` 的长度计算可发生整数溢出，使后续写入越过 BSS 缓冲区。通常 BSS 后没有可利用对象，但约有 $1/0x2000$ 的概率 heap 紧邻 BSS。程序的“next life”会通过 `execve` 重启自身并重新随机布局，因此可以在同一连接中反复尝试。

每轮先分配孩子姓名，再释放但保留悬空的 `lastName` 指针，读取资料即可泄露 heap 低位。官方 solver 在泄露值为 `0x405` 时认定布局满足相邻条件：

```python
while True:
    next_life()       # execve
    adopt_kid()
    disown_kid()      # free 后指针未清除
    leak = show_stats()
    if leak == 0x405:
        break
```

### 覆写 tcache 并获得任意分配

触发整数溢出准备超长 BSS 写入，跨过边界覆盖 heap 上的 `tcache_perthread_struct`。伪造某个 bin 的计数和 entries 指针后，下一次对应大小的 `malloc` 会返回攻击者指定地址。

先让分配落到全局 `profile` 结构，改写 `yourName` 指针指向 GOT，借资料显示功能泄露 libc；再指向 libc 的 `environ` 获得栈地址。关键链条是：

```text
BSS overflow
  -> overwrite tcache entries
  -> malloc at profile/GOT
  -> leak libc
  -> malloc at environ
  -> leak stack
```

### 写入栈上 ROP

再次伪造 tcache entry 指向泄露栈地址附近，使下一次分配覆盖返回路径。ROP 先用 `ret` 对齐，再调用 `system("/bin/sh")`。取得 shell 后读取 flag：

```text
grey{not_every_life_is_equal_you_are_one_in_eight_thousand_one_hundred_and_ninety_two!}
```

## 方法总结

该题的核心不是单一堆技巧，而是把多个有限原语串联：`execve` 提供可重复 ASLR 抽样，UAF 提供布局判据，BSS 溢出控制 tcache，任意分配再完成 libc/栈泄露和 ROP。任何硬编码偏移都依赖题目附带二进制与 libc，复现时应从对应文件重新计算。
