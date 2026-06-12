# nestDLP

## 题目简述

题目把 flag 去掉 `LilacCTF{}` 后作为明文 `m`，每次生成一个特殊 padding 与 `m` 异或，得到 384 个指数 `e`。随后在模 `p^3` 的二元多项式商环中输出 `g^e`：

```python
R = PolynomialRing(Zmod(p**3), names="x,y")
I = R.ideal([
    x**3 + y**5 + 13 * x * y - 37,
    y**3 + x**5 + 37 * x - 13,
])
S = R.quotient(I, names=("x,y"))
g = S(x**2 + y**2 + 13 * x + 37 * y + 1337)
print(g**e)
```

每个 padding 的二进制串有固定汉明重量：左半部分全是 `0`，右半部分全是 `1`，打乱后再与明文异或。题目由两层构成：先从商环离散对数恢复所有指数，再利用固定汉明重量约束恢复明文。

## 解题过程

### 商环离散对数

直接在二元多项式商环里做 DLP 很难，但可以把“乘以某个商环元素”表示成基上的线性变换矩阵。对 `g` 和输出元素 `h = g^e` 分别构造乘法矩阵后，有：

$$
\det(M_h) = \det(M_g)^e
$$

于是问题被降维成 `Zmod(p^3)` 上的 p-adic DLP。求解逻辑为：

```python
det_g = get_matrix(g, S, p).det()
det_h = get_matrix(h, S, p).det()
log_det = padic_dlp(det_g, det_h, p, 3)
exps.append(log_det)
```

`padic_dlp` 使用：

```python
theta(k) = (k^((p - 1) * p^(s - 1)) - 1) // p^s
```

把指数恢复到模 `p^(s-1)` 的范围。由于真实指数长度远小于这个范围，可以直接得到每次 `m ^ padding` 的值。

### 汉明重量约束恢复明文

padding 的构造是：

```python
left = ["0"] * (half_len - 1)
right = ["1"] * (half_len + 1)
bits = left + right
shuffle(bits)
padding = int("".join(bits), 2)
```

也就是说，对每个恢复出的 `c = m ^ padding`，`m ^ c` 的二进制中恰有 `half_len + 1` 个 `1`。把明文每一位作为布尔变量，就能为每条输出建立一个线性等式：

```python
model.Add(sum(coeffs[j] * m[j] for j in range(bitlen)) == rhs)
```

再加上每个字节为可打印 ASCII 的约束，使用 OR-Tools CP-SAT 即可恢复 flag 内部字符串。

### 验证

求解脚本会先从所有输出中恢复指数列表，再把指数作为约束输入 CP-SAT。恢复出的内部字符串拼回 `LilacCTF{...}` 后即可作为最终结果。

## 方法总结

- 核心技巧：把商环元素乘法转成矩阵，用行列式把商环 DLP 化为 p-adic DLP，再用约束求解恢复明文。
- 识别信号：输出是 `g^e`，但 `g` 位于可构造有限维基的商环中，应考虑“乘法矩阵表示”。
- 复用要点：当密文是 `m xor padding` 且 padding 有全局汉明重量约束时，每组样本都能转成关于明文 bit 的线性方程，适合交给 CP-SAT 或格方法处理。
