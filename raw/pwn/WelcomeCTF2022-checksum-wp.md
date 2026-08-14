# checksum

## 题目简述

程序维护 `i64 history[16]`。读取历史时只检查 `read_idx >= 16`，没有拒绝负数；写入时则完全不检查递增的 `history_idx`。这分别形成越界读与连续越界写，但要覆盖返回地址，还必须先恢复栈 Canary 和 libc 基址。

## 解题过程

`history[12]` 中残留了一个靠近 libc 的指针，官方脚本读取它并减去固定偏移，得到配套 `libc-2.31.so` 的加载基址。

Canary 位于逻辑上的 `history[17]`，正常索引会被 `read_idx >= 16` 拦截。比较使用有符号数，而地址计算会执行 `read_idx * 8`。传入：

$$
17-2^{63}
$$

时，该值在有符号比较中为负数；乘以 8 后按 64 位回绕，地址偏移却与 $17\times8$ 相同，因此可以读出 Canary：

```python
libc.address = read_at(12) - 0x32e8 - 2023424
canary = read_at(17 - (1 << 63))
```

随后构造 `system("/bin/sh")` 的 libc ROP 链。利用无边界的 `history_idx`，依次写 17 个占位值、原 Canary、保存的 `rbp` 和 ROP 各个 8 字节字。由于输入格式是 `%lld`，高位为 1 的机器字要转换为对应的有符号十进制：

```python
def signed_qword(value):
    return value if value < 1 << 63 else value - (1 << 64)
```

最后写入 `1337` 使 `checksum()` 返回真并退出 `main`。Canary 校验通过后，函数从已覆盖的返回地址执行 ROP，得到 Shell：

```text
greyhats{M4tH_H4rD_pwN_b4ng_B4nG_32afe1}
```

## 方法总结

本题把有符号边界绕过、整数乘法回绕、未初始化栈泄露和顺序越界写组合在一起。负索引并非只能向数组前方读取；在固定宽度地址运算中，精心选择的负数可回绕到正向目标。覆盖控制流前要按真实栈布局恢复 Canary，并确保每个 64 位 ROP 字以 `%lld` 能接受的形式发送。
