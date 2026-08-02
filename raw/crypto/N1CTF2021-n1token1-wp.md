# N1CTF 2021 - n1token1

## 题目简述

题目给出 1024 位 RSA 模数 $n$ 和 920 个 token。每个 token 与同一个 RSA 密文 $c$ 满足

$$
token_i^2\equiv c^2x_i\pmod n,
$$

其中 $x_i$ 是至多约 937 位的平滑数。解法先用 LLL 从多个同余式中恢复小量 $x_i$ 与公共量 $c^2$，再仿照二次筛构造平方同余，从而分解 $n$。

官方文字把比值方向写成了 $c^2/token_i^2$，但随附代码实际计算的是 `token[i]^2 * inverse(c2, n)`；后续分解与矩阵构造也只与上式一致，因此本文以官方求解代码为准。

## 解题过程

### 用格规约恢复公共的 $c^2$

任选 20 个 token。由

$$
token_i^2\equiv c^2x_i\pmod n
$$

可消去 $c^2$，得到

$$
token_{i+1}^2x_0-token_0^2x_{i+1}\equiv0\pmod n.
$$

所有 $x_i$ 都明显小于模数，因此可以把 19 个同余关系嵌入一个 $39\times39$ 整数格，并把模约束坐标左移 1024 位以提高权重：

```python
size = 20
B = matrix(ZZ, 2*size - 1, 2*size - 1)

for i in range(size):
    B[i, i] = 1
for i in range(size - 1):
    B[0, size+i] = (token[i+1]^2 % n) << 1024
    B[i+1, size+i] = (-token[0]^2 % n) << 1024
    B[size+i, size+i] = n << 1024

C = B.LLL()
x0 = abs(C[0, 0])
c2 = token[0]^2 * inverse_mod(x0, n) % n
```

短向量的前 20 个坐标给出小的 $x_i$ 关系；取出 $x_0$ 后即可从 $token_0^2\equiv c^2x_0$ 求得公共的 `c2`。应对所有 token 检查

```python
xi = token[i]^2 * inverse_mod(c2, n) % n
```

是否确实较小且能在给定的筛基上完全分解。

### 构造平方同余

把每个平滑数写成

$$
x_i=\prod_j p_j^{a_{i,j}}.
$$

建立 $920\times921$ 的指数矩阵：前 920 列记录小素数指数，最后一列恒为 1，表示每条等式中的一个 $c^2$ 因子。将矩阵化到 $\operatorname{GF}(2)$ 后求左核：

```python
A = matrix(ZZ, 920, 921)
for i in range(920):
    xi = token[i]^2 * inverse_mod(c2, n) % n
    A[i, 920] = 1
    # 将 xi 在 sieve_base 上分解，把指数写入 A[i, :920]

relations = matrix(GF(2), A).left_kernel().basis()
```

任取非零核向量，选中的每一列指数和都是偶数。于是可分别构造

```python
X = 1
exponents = vector(ZZ, 921)
for i in range(920):
    if relations[0][i]:
        X = X * token[i] % n
        exponents += A[i]

Y = pow(c2, int(exponents[920]) // 2, n)
for j, prime in enumerate(primes):
    Y = Y * pow(prime, int(exponents[j]) // 2, n) % n
```

这时

$$
X^2\equiv Y^2\pmod n.
$$

若该平方同余不是平凡的 $X\equiv\pm Y\pmod n$，就可以分解模数：

```python
p = gcd(X - Y, n)
q = gcd(X + Y, n)
assert 1 < p < n and p*q == n
```

### 解密

求出私钥指数 $d=e^{-1}\bmod\varphi(n)$ 后，`pow(c2, d, n)` 得到明文整数的平方，取整数平方根即可还原消息：

```python
from math import isqrt
from Crypto.Util.number import long_to_bytes

d = inverse_mod(65537, (p - 1) * (q - 1))
m2 = int(pow(c2, d, n))
m = isqrt(m2)
assert m*m == m2
print(long_to_bytes(m))
```

仓库没有附带 `output.txt` 和 `sieve_base`，所以现有材料不足以重跑并确认最终 flag；题目的完整数学机制与求解流程则由官方 WP 和代码明确给出。

## 方法总结

本题把两种技术串在一起：LLL 利用“模数很大、隐藏乘数较小”恢复公共关系，GF(2) 左核则像二次筛一样把若干平滑关系乘成平方。判断公式方向时不能只依赖文字描述，应以求解脚本的实际不变量和后续矩阵含义交叉验证。
