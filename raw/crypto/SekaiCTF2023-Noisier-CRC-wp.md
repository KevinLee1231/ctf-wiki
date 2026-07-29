# Noisier CRC

## 题目简述

本题延续 Noisy CRC，但每次查询从 3 个候选增加到 13 个，并强制生成多项式必须是 16 次不可约多项式；总查询次数限制为 133。秘密仍是 512 位整数，CRC 定义仍为：

$$
\operatorname{CRC}_f(\text{key})
=\text{key}\cdot x^{16}\bmod f(x).
$$

若直接沿用上一题的表示，每个模数会贡献 13 个 CRT 向量，环境维数为 $16n$，候选空间最多为 $13n$ 维，只剩 $3n$ 个维度约束；在 $n\le133$ 时不足以唯一限制 512 位秘密。

关键是选定每组的一个基准候选，把每组 13 个绝对向量改写成 12 个差分向量。

## 解题过程

### 构造 CRT 候选项

选择 133 个互不相同的 16 次不可约多项式 $f_i$，并按 Noisy CRC 的方法计算：

$$
A_{i,j}
=\left(c_{i,j}(M_i x^{16})^{-1}\bmod f_i\right)M_i,
\qquad M_i=M/f_i.
$$

正确密钥满足：

$$
\text{key}=\bigoplus_i A_{i,j_i}.
$$

### 以每组第一个候选为基准

令：

$$
B=\bigoplus_i A_{i,0}.
$$

在等式两侧同时异或 $B$：

$$
\text{key}\oplus B
=\bigoplus_i\left(A_{i,j_i}\oplus A_{i,0}\right).
$$

当 $j_i=0$ 时差分为零；其余只可能落在每组 12 个非零向量中。因此右侧属于至多 $12n$ 个向量张成的空间，而不是原来的 $13n$ 维空间。

实现时一边构造基准和，一边收集差分：

```python
baseline = 0
differences = []

for f, candidates in responses.items():
    terms = build_crt_terms(f, candidates, modulus_product)
    baseline ^= terms[0]
    differences.extend(term ^ terms[0] for term in terms)
```

### 消去高位得到唯一低次数代表

环境空间维数为：

$$
16n=2128,
$$

差分空间最多为：

$$
12n=1596.
$$

二者相差：

$$
4n=532>512.
$$

这意味着陪集 `baseline + span(differences)` 中预期存在唯一一个次数低于 512 的元素，即真实 `key`。先对差分向量做最高位高斯消元，再用基向量消掉 `baseline` 的所有高位：

```python
basis = gaussian_basis(differences, width=16 * 133)

key = baseline
for bit in range(16 * 133 - 1, -1, -1):
    if basis[bit] is not None and ((key >> bit) & 1):
        key ^= basis[bit]
```

所得 `key` 再经：

```python
aes_key = sha256(long_to_bytes(key)).digest()[:16]
flag = AES.new(
    aes_key, AES.MODE_CTR, nonce=b"12345678"
).decrypt(encrypted_flag)
```

即可恢复明文。

## 方法总结

- 核心技巧：为每组候选选择一个基准，把 $13n$ 个绝对 CRT 向量降为 $12n$ 个差分向量，使剩余余维从 $3n$ 增加到 $4n$。
- 识别信号：上一版线性空间攻击几乎可行，但候选数增加后只差一个线性维度因子；所有候选又天然按查询分组。
- 复用要点：处理“每组恰选一个向量”的问题时，先减去或异或固定基准，通常能消掉每组一个自由度。最终不是求任意线性组合，而是求某个陪集中满足低次数或短向量约束的唯一代表。
