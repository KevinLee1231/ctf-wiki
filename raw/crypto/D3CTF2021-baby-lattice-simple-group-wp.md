# babyLattice & simpleGroup

## 题目简述

`babyLattice` 和 `simpleGroup` 在官方 WP 中合并讲解，二者使用同一类基于 RSA 模数分解和小矩阵的公钥方案。私钥包含 RSA 模数 `n` 的分解和一个元素较小的 2 阶矩阵 `A`；公钥由 CRT 提升后的列向量比值 `b = a1 * a2^(-1) mod n` 得到。加密形如 `c = b*m + r mod n`，其中消息和随机数都较小。

`babyLattice` 是明文恢复：寻找与 `b` 等价但更小的 `(a1', a2')`，把模 `n` 方程转回整数小关系后用 LLL 解出 `m`。`simpleGroup` 是密钥恢复：回到 key generation 构造低维格，恢复关于秘密比值的二次多项式，因式分解后通过 `gcd` 拿到 `n` 的因子，再解线性方程恢复消息。

## 解题过程

题目脚本和 exp 见 LurkNoi 的公开 gist：<https://gist.github.com/LurkNoi/c42dd9379f2070830f7d973f7862ef29>。下面保留的是该脚本背后的参数结构、LLL 格构造和密钥恢复公式。

方案可整理为：

- 私钥：`n = p*q` 的分解，其中 `n` 为 1024 bit；以及元素约 100 bit 的小矩阵 `A = [[a11, a12], [a21, a22]]`。
- 公钥：把矩阵列向量通过 CRT 提升为 `a1, a2`，发布 `b = a1 * a2^(-1) mod n`。
- 加密：消息 `m` 至多约 400 bit，随机选择约 400 bit 的 `r`，计算 `c = b*m + r mod n`。
- 解密依据：在已知私钥时，`a2*c mod p = a11*m + a12*r` 这类值足够小，可以转成整数方程求解。

密钥生成时先取 RSA 模数 `n = p*q`，再取小元素矩阵 `A`。把 `A` 的两列分别经 CRT 提升到模 `n` 下得到 `a1, a2`，公钥发布 `b = a1 * inverse(a2, n) mod n`。加密时消息 `m` 和随机数 `r` 都远小于 `n`，密文为 `c = b*m + r mod n`。

解密成立的原因是：私钥知道 `p/q` 和矩阵 `A`，可以把 `a2*c` 分别约化到 `p`、`q` 上，得到形如 `a11*m + a12*r`、`a21*m + a22*r` 的小整数关系，再解线性方程恢复 `m`。

plaintext-recovery attack

明文恢复的关键观察是 `b` 只定义了一个模 `n` 下的比值。若能找到更小的等价表示 `(a2', a1')`，满足 `a1' / a2' == b (mod n)`，就可以把 `a2' * c = a1' * m + a2' * r (mod n)` 近似视为整数上的小关系。

This can be easily solved by LLL algorithm with basis

明文恢复使用二维格：

```python
M = Matrix(ZZ, [
    [1, b],
    [0, n],
])
```

LLL 找到短向量后可得到约 512 bit 的 `(a2', a1')`，进而利用 `a2' * c mod n = a1' * m - (-a2') * r` 求 `m`。

LLL 约化出的约 512 bit 短向量可能符号不同，但不影响后续求解。

```python
M = Matrix(ZZ, [
    [1, b],
    [0, n],
])
B = M.LLL()
aa2, aa1 = B[0]  # about 512 bits, usually aa2 < 0 < aa1

# aa2 * c % n = aa1*m - (-aa2)*r
s = c * aa2 % n
m = s * inverse_mod(aa1, -aa2) % (-aa2)
```

另一类非预期同样来自 `c = b*m + r mod n` 中 `m/r` 足够小：把 `b*m + r - c` 作为短关系构造更高维格，也可以直接还原明文。

key-recovery attack

密钥恢复要回到 key generation。因为公钥比值 `b` 来自两列向量 CRT 提升后的比值，分别在两个素数模数上有：

```text
a12*b - a11 = 0 mod p
a22*b - a21 = 0 mod q
```

两式相乘可得到关于秘密比值的二次多项式：

```text
F(x) = (a12*x - a11) * (a22*x - a21)
```

该多项式在 `Z/nZ` 上有一个根等于公钥比值 `b`。若写成 `F(x) = c2*x^2 + c1*x + c0`，由于 `ci` 足够小，可以构造三维格恢复这些小系数：

密钥恢复使用三维格：

```python
M = Matrix(ZZ, [
    [1, 0, -b**2],
    [0, 1, -b],
    [0, 0, n],
])
```

LLL 给出二次多项式 `c2*x^2 + c1*x + c0` 的小系数。因式分解后得到 `a11/a12` 和 `a21/a22` 两个秘密比值。由上面的同余式可知，`b - a11/a12` 会与公开模数 `n` 共享非平凡因子，因此用 `gcd(frac % n - b, n)` 即可恢复 `n` 的素因子。两个素因子的顺序不影响最终解密；若顺序取反，矩阵行也相应交换，解出的消息仍一致。

```python
M = Matrix(ZZ, [
    [1, 0, -b**2],
    [0, 1, -b],
    [0, 0, n],
])
B = M.LLL()
c2, c1, c0 = B[0]

PR = PolynomialRing(QQ, "x")
x = PR.gen()
f = c2*x**2 + c1*x + c0
factors = f.factor()

primes = []
A = []
for term, _ in factors:
    frac = -term(x=0)
    A.append([frac.numer(), frac.denom()])
    p = gcd(frac % n - b, n)
    assert 1 < p < n
    primes.append(p)

A = Matrix(ZZ, A)
a2 = crt([A[0, 1], A[1, 1]], primes)
s1 = a2 * c % primes[0]
s2 = a2 * c % primes[1]
m, r = A.solve_right(vector([s1, s2]))
flag = "d3ctf{%s}" % sha256(int(m).to_bytes(50, "big")).hexdigest()
```

## 方法总结

- 核心技巧：利用小参数把模 `n` 上的线性/二次关系转成低维格短向量问题；明文恢复用二维 LLL，密钥恢复用三维 LLL 加多项式分解。
- 识别信号：RSA 模数下公开值是秘密小向量比值，密文形如 `b*m+r`，且 `m/r` 位数明显小于 `n`。
- 复用要点：这类题要先找“模关系中的小整数代表元”；若能找到更小的等价 `b` 表示，就可以把取模方程还原为整数方程。

