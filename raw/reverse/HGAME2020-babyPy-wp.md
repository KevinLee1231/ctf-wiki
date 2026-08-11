# babyPy

## 题目简述

附件只暴露了 Python 字节码，需要先用 `dis` 还原加密函数。算法把输入反转后做相邻异或，并输出十六进制。该变换可从数组尾部向前原地逆转。

## 解题过程

根据字节码中的 `BUILD_SLICE`、`BINARY_XOR` 和循环结构，可还原：

```python
def encrypt(data):
    reversed_data = data[::-1]
    values = list(reversed_data)
    for index in range(1, len(values)):
        values[index] = values[index - 1] ^ values[index]
    return bytes(values).hex()
```

设变换后的序列为 $c$、反转后的原序列为 $r$，则：

$$
c_0=r_0,\qquad c_i=c_{i-1}\oplus r_i.
$$

因此 $r_i=c_i\oplus c_{i-1}$。必须从末尾向前原地计算，避免先覆盖仍要使用的 $c_{i-1}$：

```python
cipher = bytes([
    0x7d, 0x03, 0x7d, 0x04, 0x57,
    0x17, 0x72, 0x2d, 0x62, 0x11,
    0x4e, 0x6a, 0x5b, 0x04, 0x4f,
    0x2c, 0x18, 0x4c, 0x3f, 0x44,
    0x21, 0x4c, 0x2d, 0x4a, 0x22,
])

values = list(cipher)
for index in range(len(values) - 1, 0, -1):
    values[index] ^= values[index - 1]

print(bytes(values[::-1]).decode())
```

输出：

```text
hgame{sT4cK_1$_sO_e@Sy~~}
```

## 方法总结

- 核心技巧：先从 Python 字节码恢复数据流，再逆向“反转 + 前缀异或”。
- 识别信号：若循环中当前项与前一项异或并覆盖当前项，通常可以从数组尾部逆推。
- 复用要点：原地逆算法的遍历方向决定正确性；从前向后会把密文前项提前改成明文，破坏后续计算。

> 原 PDF 没有给出完整密文字节；公开解题记录补足了常量，本文已独立运行逆算法确认结果。参考：[HGame 2020 Week2 题解](https://rinko.work/2020/02/01/hgame_week2/)。
