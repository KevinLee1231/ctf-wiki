# alkane

## 题目简述

题目用 flag 花括号内的 16 字节作为密钥，对已知明文 `yellow submarine` 加密，并公开完整的 `schedule` 与密文。所谓密钥调度并没有引入非线性：每个输出位只是若干输入密钥位的异或；随后加密又把已知明文与调度结果异或。因此整道题可以写成 GF(2) 上的线性方程组。

## 解题过程

设 128 个密钥位组成列向量 $x$。对第 $i$ 个输出位，`schedule[i]` 给出的下标集合表示

$$
y_i=\bigoplus_{j\in S_i}x_j.
$$

又因为密文 $c=p\oplus y$，已知 `yellow submarine` 后可以直接计算 $y=p\oplus c$。于是每个输出位提供一条线性方程，把系数和常数项写成 128 行、129 列的增广矩阵，再进行 GF(2) 高斯消元。

```python
def bits(data: bytes):
    return [(byte >> (7 - bit)) & 1
            for byte in data for bit in range(8)]

def pack_bits(value: int) -> bytes:
    out = bytearray(16)
    for index in range(128):
        out[index // 8] |= ((value >> index) & 1) << (7 - index % 8)
    return bytes(out)

def solve_linear(schedule, plaintext: bytes, ciphertext: bytes):
    rhs = bits(bytes(a ^ b for a, b in zip(plaintext, ciphertext)))
    rows = []
    for indexes, value in zip(schedule, rhs):
        mask = 0
        for index in indexes:
            mask ^= 1 << index
        rows.append([mask, value])

    pivot_for_col = {}
    pivot_row = 0
    for col in range(128):
        found = next((r for r in range(pivot_row, 128)
                      if (rows[r][0] >> col) & 1), None)
        if found is None:
            continue
        rows[pivot_row], rows[found] = rows[found], rows[pivot_row]
        for r in range(128):
            if r != pivot_row and ((rows[r][0] >> col) & 1):
                rows[r][0] ^= rows[pivot_row][0]
                rows[r][1] ^= rows[pivot_row][1]
        pivot_for_col[col] = pivot_row
        pivot_row += 1

    free = [c for c in range(128) if c not in pivot_for_col]
    for choice in range(1 << len(free)):
        x = [(choice >> i) & 1 for i in range(len(free))]
        value = sum(bit << col for bit, col in zip(x, free))
        for col, row in reversed(list(pivot_for_col.items())):
            parity = (rows[row][0] & value).bit_count() & 1
            value |= (rows[row][1] ^ parity) << col
        yield pack_bits(value)
```

实际矩阵秩为 127，只有一个自由变量，所以仅有两个候选。一个候选含不可打印字节，另一个是可读密钥：

```text
l1neeruwu8215413
```

补上题目格式得到：

```text
tjctf{l1neeruwu8215413}
```

## 方法总结

- XOR 组合天然位于 GF(2)；只要没有 S 盒、乘法等非线性环节，就应优先检查能否直接列线性方程。
- 方程组不满秩并不等于无法恢复。自由变量很少时，枚举全部解，再用可打印字符和 flag 格式筛选即可。
- 实现时必须统一位序。最稳妥的做法是严格复刻题目中“按字节、按位”的展开顺序，并用已知密文重新加密候选作最终验证。
