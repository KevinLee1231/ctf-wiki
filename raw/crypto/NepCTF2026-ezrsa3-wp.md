# NepCTF2026 ezRSA3 Writeup

## 题目简述

题目生成 RSA 素因子时，从公开的 10000 个约 50 位素数 `sops` 中随机选 10 个，并令：

$$
p=2\prod_{j=1}^{10}r_j-1.
$$

另一个因子 $q$ 为普通 512 位素数。由于：

$$
p+1=2\prod_{j=1}^{10}r_j,
$$

$p+1$ 的全部素因子都来自公开集合，适合使用 Williams $p+1$ 因子分解法。

## 解题过程

Lucas 序列 $V_k(P,1)$ 定义为：

$$
V_0=2,\qquad V_1=P,\qquad
V_k=P V_{k-1}-V_{k-2}.
$$

当参数 $P$ 的判别式满足相应二次非剩余条件，且指数 $M$ 是 $p+1$ 的倍数时，有：

$$
V_M(P,1)\equiv 2\pmod p.
$$

于是：

$$
\gcd(V_M(P,1)-2,N)
$$

会泄露 $p$。这里可以令：

$$
M=2\prod_{r\in\text{sops}}r.
$$

无需显式构造这个巨大整数。利用 Lucas 序列的复合性质：

$$
V_{ab}(P,1)=V_a(V_b(P,1),1),
$$

依次对每个公开素数更新：

```python
V = lucas_V(2, P, N)
for index, r in enumerate(sops):
    V = lucas_V(r, V, N)
    if index % 500 == 499:
        g = gcd(V - 2, N)
        if 1 < g < N:
            p = g
            break
```

`lucas_V` 应用二进制倍增实现，使每个指数 $r$ 只需 $O(\log r)$ 次模运算。并非每个 $P$ 都满足目标素因子上的判别式条件，因此依次尝试若干小参数，例如 `3, 4, 5, 7, 11, ...`；若 `gcd` 得到 $1$ 或 $N$，更换 $P$ 重新执行。

得到非平凡因子后，按普通 RSA 解密：

```python
q = N // p
phi = (p - 1) * (q - 1)
d = inverse_mod(e, phi)
m = pow(c, d, N)
flag = int(m).to_bytes((int(m).bit_length() + 7) // 8, "big")
print(flag.decode())
```

结果为：

```text
NepCTF{5m0o7h_m4k3s_w1lliam_gr34t}
```

还可验证恢复出的 $p$ 满足 `(p + 1) // 2` 恰好被 `sops` 中 10 个元素整除，从而确认因子并非偶然碰撞。

## 方法总结

看到 RSA 素因子的 $p-1$ 或 $p+1$ 具有公开光滑结构时，应分别联想到 Pollard $p-1$ 与 Williams $p+1$。本题公开全部候选素因子，使 Williams 法可以用 Lucas 复合逐步吸收指数；定期计算 gcd 既能提前命中，也避免一次构造巨大的 $M$。
