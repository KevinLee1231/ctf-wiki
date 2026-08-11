# Inv

## 题目简述

题目把 0 至 255 的排列作为替换表，并用替换表复合定义乘法 `Mul`。这些排列在复合运算下构成群，单位元为 `bytes(range(256))`。已知生成元 $s$ 的 $s^{595}$、$s^{739}$ 和用 $s$ 替换后的密文，目标是利用扩展欧几里得算法求出 $s^{-1}$。

## 解题过程

把替换和复合写成：

```python
IDENTITY = bytes(range(256))

def subs(table: bytes, data: bytes) -> bytes:
    return bytes(table[value] for value in data)

def mul(left: bytes, right: bytes) -> bytes:
    return subs(left, right)

def inv(table: bytes) -> bytes:
    return bytes(table.find(bytes([value])) for value in range(256))

def power(base: bytes, exponent: int) -> bytes:
    result = IDENTITY
    while exponent:
        if exponent & 1:
            result = mul(result, base)
        base = mul(base, base)
        exponent >>= 1
    return result
```

扩展欧几里得算法给出：

$$
1=-272\times595+219\times739.
$$

等价地：

$$
-1=272\times595-219\times739.
$$

因此：

$$
s^{-1}=(s^{595})^{272}\cdot\left((s^{739})^{-1}\right)^{219}.
$$

代入题目给出的两个排列和密文：

```python
s_inverse = mul(
    power(s595, 272),
    power(inv(s739), 219),
)
flag = subs(s_inverse, cipher)
print(flag.decode())
```

得到：

```text
hgame{U_kN0w~tH3+eXtEnD-EuC1iD34n^A1G0rIthM}
```

题目中由 $s$ 生成的子群阶为 48720，也可先求相关元素的阶，再把指数按群阶化简；扩展欧几里得路线不需要恢复原始替换表，结构更直接。

## 方法总结

- 核心技巧：把字节替换识别为置换群运算，用 Bézout 系数把两个已知幂组合成 $s^{-1}$。
- 识别信号：运算满足结合律、存在单位元和逆元，且给出同一元素的两个互素指数幂时，与 RSA common modulus attack 的指数消元思路相同。
- 复用要点：先确认 `Mul` 的复合方向；若左右顺序写反，即使指数关系正确，也可能在非交换群中得到错误结果。

> 公开 Crypto 复盘补足了原 PDF 未展示的最终 flag，正文已保留完整群运算与指数推导。参考：[HGame 2020 Crypto 题解](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/)。
