# eezzDLP

## 题目简述

题目延续矩阵离散对数，但这次模数是 `n = p^2`，矩阵 `a` 被特殊构造为行列式为 1，因此上一题通过 `det(b)=det(a)^k` 降维的方法失效。附件脚本仍然生成 `b = a ** k`，并用 `md5(long_to_bytes(k))` 作为 AES-ECB 密钥加密 flag。

题目定义代码的关键部分如下：

```python
p = get_random_prime()
q = get_random_prime()
n = p * p

a = Matrix(Zmod(n), getRandomMatrix())
k = getPrime(660)
b = a ** k
save([n, a, b], "data.sobj")

key = hashlib.md5(long_to_bytes(k)).digest()
ciphertext = AES.new(key, AES.MODE_ECB).encrypt(flag)
```

由于 `det(a)=1`，需要改用矩阵特征值。若 `lambda`、`mu` 分别是 `a`、`b` 的对应特征值，则有 `mu = lambda^k`，可以在 p-adic 环上恢复大部分 `k`。

## 解题过程

先加载 `n, a, b`，由 `n = p^2` 得到 `p = sqrt(n)`。把特征多项式换到 `Zp(p, prec=2)` 上，取根作为特征值：

```python
n, a, b = load("data.sobj")
p = Integer(n).sqrt()
assert p * p == n

P = Zp(p, prec=2)
f_A = a.charpoly().change_ring(ZZ).change_ring(P)
f_B = b.charpoly().change_ring(ZZ).change_ring(P)

la = f_A.roots(multiplicities=False)[0]
mu = f_B.roots(multiplicities=False)[0]
```

在 p-adic 环中计算：

```python
order = p - 1
k_high = int(((mu ** order).log() / (la ** order).log()).lift())
```

这一步能恢复约 608 bit 信息。再利用 `p - 1` 中的小素因子 `sm_p = 688465747867`，对 `p^2` 上的特征值做 Pohlig-Hellman，补充约 40 bit：

```python
sm_p = 688465747867
low = pohlig_hellman(
    pow(int(la.lift()), p * ((p - 1) // sm_p), p**2),
    pow(int(mu.lift()), p * ((p - 1) // sm_p), p**2),
    p**2,
    sm_p,
)

res = crt([k_high, low], [p, sm_p])
```

剩余只有少量 bit，直接枚举：

```python
for i in range(2**10):
    k = res + i * p * sm_p
    if a ** k == b:
        break
```

由于两个矩阵的特征值顺序不一定对应，`f_A` 的两个根都要和 `f_B` 的根尝试组合。恢复 `k` 后，用 `md5(long_to_bytes(k))` 解 AES-ECB 密文。

## 方法总结

- 核心技巧：行列式失效时改用矩阵特征值，把 `a ** k = b` 转成特征值上的 DLP，并用 p-adic log 恢复高位。
- 识别信号：矩阵 DLP 中 `det(a)=1` 或行列式不给信息时，应检查特征值、Jordan 结构或其他矩阵不变量。
- 复用要点：特征值匹配可能存在顺序问题，实战中要枚举根的对应关系，并结合原矩阵幂等式校验最终 `k`。
