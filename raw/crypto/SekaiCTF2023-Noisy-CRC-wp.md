# Noisy CRC

## 题目简述

服务随机生成一个 512 位 `key`，以 `SHA256(key)[:16]` 作为 AES-CTR 密钥加密 flag。玩家可以重复提交不同的 16 次生成多项式 $f(x)$，每次获得三个 16 位候选值，其中恰好一个是：

$$
\operatorname{CRC}_f(\text{key})
=\text{key}\cdot x^{16}\bmod f(x).
$$

所有运算都在 $\operatorname{GF}(2)[x]$ 中进行。噪声使每个查询看似有三种余数，逐项枚举需要 $3^n$ 次组合；但中国剩余定理把每种组合写成若干固定向量的异或和，因此可以转成线性代数问题。

## 解题过程

### 选择两两互素的模多项式

选择 $n$ 个互不相同的 16 次不可约多项式：

$$
f_1,f_2,\ldots,f_n.
$$

不可约且互异保证它们两两互素。令：

$$
M=\prod_{i=1}^{n}f_i,\qquad M_i=M/f_i.
$$

对每个 $f_i$，服务返回三个候选余数 $c_{i,0},c_{i,1},c_{i,2}$。

### 构造每个候选的 CRT 基向量

CRC 余数对应的是 $\text{key}\cdot x^{16}$，因此需要求：

$$
e_i=(M_i x^{16})^{-1}\pmod{f_i}.
$$

每个候选对应的 CRT 项为：

$$
A_{i,j}=(c_{i,j}e_i\bmod f_i)M_i.
$$

若 $j_i$ 恰好选择了第 $i$ 次查询中的真实余数，则：

$$
\text{key}\equiv\bigoplus_{i=1}^{n}A_{i,j_i}\pmod M.
$$

当 $\deg M=16n>512$ 时，低于 512 次的 `key` 不会再发生模 $M$ 回绕。

对应实现：

```python
terms = []
M = product_over_gf2(moduli)

for f, candidates in responses.items():
    Mi, _ = poly_div(M, f)
    inv = inverse_mod(poly_mul_mod(Mi, 1 << 16, f), f)

    terms.extend(
        poly_mul(poly_mul_mod(candidate, inv, f), Mi)
        for candidate in candidates
    )
```

### 用高斯消元代替 $3^n$ 枚举

所有可能的正确密钥都位于 $3n$ 个向量 $A_{i,j}$ 张成的 $\operatorname{GF}(2)$ 线性空间中。环境空间维数为 $16n$，而密钥只占低 512 位。

把全部候选项按最高位做二进制高斯消元：

```python
basis = [None] * (16 * n)

for value in terms:
    for bit in range(16 * n - 1, -1, -1):
        if not (value >> bit) & 1:
            continue
        if basis[bit] is None:
            basis[bit] = value
            break
        value ^= basis[bit]
```

官方脚本使用 64 个模多项式，此时环境维数为 1024、候选向量最多 192 个。消元后检查最高位低于 512 的非零基向量，把每个向量当作候选 `key` 尝试 AES-CTR 解密；正确结果可由 `SEKAI{...}` 格式立即确认：

```python
for bit in range(512):
    if basis[bit] is None:
        continue
    candidate = basis[bit]
    aes_key = sha256(long_to_bytes(candidate)).digest()[:16]
    plaintext = AES.new(
        aes_key, AES.MODE_CTR, nonce=b"12345678"
    ).decrypt(encrypted_flag)
    if plaintext.startswith(b"SEKAI{"):
        print(plaintext)
```

根据维数估计，$16n-3n=13n>512$ 时通常已经足够，约 40 次查询即可；官方实现取 64 次以提高余量。

## 方法总结

- 核心技巧：把每次“三选一”的 CRC 余数通过多项式 CRT 提升为固定向量，再在 $\operatorname{GF}(2)$ 上求低次数线性组合。
- 识别信号：攻击者可选择两两互素的生成多项式，同一低次数秘密在每个模数下泄漏少量候选余数。
- 复用要点：CRC 对应多项式余数，普通整数乘法、除法和加法都应分别换成移位异或、多项式长除和 XOR。构造 CRT 项时不要漏掉 CRC 预先乘上的 $x^{16}$。
