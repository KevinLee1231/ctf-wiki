# miniLCTF 2024 Sums Writeup

## 题目简述

密钥生成先选取 256 个 128 位整数 $s_i$，令

$$p=\sum_i s_i+2$$

且要求 $p$ 为素数；随后随机选择 $e$，公开 $a_i=e s_i\bmod p$ 和 $b_i=s_i\bmod2$。加密每一位时选择满足奇偶校验的二进制向量 $r$，输出

$$c=\sum_i a_i r_i.$$

目标是从公钥恢复 $p,e$，再逐位解密 secret。

## 解题过程

### 利用素数生成条件构造格

令 $A=\sum_i a_i$。因为 $a_i=e s_i-k_i p$，且 $\sum_i s_i=p-2$，有

$$A=e(p-2)-Kp,$$

从而导出同时约束 $p$、$e$ 和各个小量 $s_i$ 的整数关系。官方解法把 255 组相对关系嵌入一个 $511\times511$ 的格：第一类坐标承载至多 128 位的 $s_i$，第二类坐标强制模关系消为零。缩放矩阵使用

```python
Q = diagonal_matrix(
    ZZ,
    [1] + 255 * [2**8] + 255 * [2**2048],
)
```

其中极大的 $2^{2048}$ 权重迫使有效短向量在最后 255 个坐标上全为零。

核心恢复过程如下：

```python
from sage.all import Matrix, ZZ, diagonal_matrix, gcd, inverse_mod, is_prime

n = len(a)
A = sum(a)
M = Matrix(ZZ, 2*n - 1, 2*n - 1)
M[0, 0] = 1

for i in range(n - 1):
    M[0, n+i] = -2*a[i+1]
    M[n+i, n+i] = 2*a[0]
    M[1+i, 1+i] = 1
    M[1+i, n+i] = A

Q = diagonal_matrix(ZZ, [1] + (n-1)*[2**8] + (n-1)*[2**2048])
reduced = (M * Q).LLL() / Q

for vec in reduced:
    if list(vec[n:]) != [0] * (n - 1):
        continue
    x0 = ZZ(vec[0])
    upper = (2*max(a) + A*2**128) // max(a)
    for candidate in list(range(x0, upper, a[0])) + list(range(-x0, upper, a[0])):
        g = gcd(candidate, A)
        if gcd(2*a[0], A) % g:
            continue
        p = inverse_mod(candidate // g, A // g) * (2*a[0] // g) % (A // g)
        if p == 2 or not is_prime(p):
            continue
        e = inverse_mod(2, p) * (-A) % p
        s_vec = [inverse_mod(e, p) * ai % p for ai in a]
        if sum(s_vec) + 2 == p and max(s_vec) < 2**128:
            break
```

本地完成 511 维 LLL 后得到：

```text
p = 44533335175712574956827555064131060303069
e = 32585490178993671429263360763435077722742
```

### 逐位解密

因为 $a_i=e s_i\bmod p$，密文满足

$$e^{-1}c\equiv\sum_i s_i r_i\pmod p.$$

而 $p=\sum_i s_i+2$，该子集和严格小于 $p$，所以模约简不会改变其奇偶性；明文位就是 $(e^{-1}c\bmod p)\bmod2$。

```python
bits = [int(inverse_mod(e, p) * ci % p) & 1 for ci in cipher]
value = int("".join(map(str, bits)), 2)
secret = value.to_bytes((value.bit_length() + 7) // 8, "big")
print(secret)
```

恢复出：

```text
S3_in_un4_n0tte_d'inv3rn0_un_vi4ggi4t0r3!@#$%^123456
```

因此完整 flag 为：

```text
miniLCTF{S3_in_un4_n0tte_d'inv3rn0_un_vi4ggi4t0r3!@#$%^123456}
```

## 方法总结

生成器看似公开了模 $p$ 的随机倍数，实际却把素数定义为所有小秘密的和再加 2，制造了很强的全局短向量关系。格恢复阶段应同时利用 128 位上界和“尾部关系坐标必须为零”；得到 $p,e$ 后，密文的奇偶性直接泄露每个明文位。
