# SU_rsa

## 题目简述

题目生成 512-bit 素数 $p,q$、256-bit 素数公钥指数 $e$，并公开 $n=pq$ 与私钥指数 $d$ 的高 512 位：

```python
d = inverse(e, (p - 1) * (q - 1))
d_m = (d >> 512) << 512
```

flag 内容是 `sha256(str(p).encode()).hexdigest()[:32]`。关键不是直接恢复 $d$ 的低位，而是由 $d$ 的高位确定 RSA 关系

$$
ed-1=k\varphi(n)=k(n-p-q+1)
$$

中的小整数 $k$，进而得到 $p\bmod e$，最后用 Coppersmith 小根恢复完整因子。

## 解题过程

### 从高位近似确定 $k$

未知部分满足 $0\le d-d_m<2^{512}$，所以乘以 256-bit 的 $e$ 后，误差仍远小于 1024-bit 的 $n$。按官方解法可直接得到：

```python
k = (e * d_m - 1) // n + 1
```

由

$$
k(n-p-q+1)+1=ed
$$

两边模 $e$，得到

$$
p+q\equiv n+1+k^{-1}\pmod e.
$$

记右侧为 $s$。又因为 $pq=n$，所以 $p\bmod e$ 与 $q\bmod e$ 正是多项式

$$
X^2-sX+n\equiv0\pmod e
$$

的两个根：

```sage
s = (n + 1 + inverse_mod(k, e)) % e
R.<x> = PolynomialRing(Zmod(e))
p0 = Integer((x^2 - s*x + n).roots()[0][0])
```

两个根分别对应 $p$ 和 $q$，任选一个尝试即可；若后续无解，再换另一个根。

### 用已知模 $e$ 的余数恢复 $p$

写成

$$
p=p_0+et.
$$

$p$ 约为 512 bit、$e$ 约为 256 bit，因此 $t$ 约为 256 bit。把 $t$ 的高 6 位枚举掉：

$$
t=x+2^{250}i,\qquad 0\le i<2^6,\quad |x|<2^{250}.
$$

对每个 $i$ 构造一次线性小根问题：

```sage
S.<x> = PolynomialRing(Zmod(n))
for i in range(2^6):
    f = e * (x + 2^250 * i) + p0
    roots = small_roots(f, X=2^250, beta=0.48, m=25)
    for r in roots:
        candidate = Integer(e * (r + 2^250 * i) + p0)
        if candidate > 1 and n % candidate == 0:
            p = candidate
            q = n // p
            break
```

这里的小根不是要求 $f(x)\equiv0\pmod n$，而是要求 $f(x)$ 与 $n$ 的一个约 512-bit 未知因子 $p$ 同余为零，因此设置 $\beta\approx0.48$。找到候选后必须以 `n % p == 0` 做精确验证。

最后按题目规则计算：

```python
flag = "SUCTF{" + sha256(str(p).encode()).hexdigest()[:32] + "}"
```

仓库附件给出的验证结果为：

```text
SUCTF{c1864501fab1841178177d4f15af4ad8}
```

## 方法总结

- 核心技巧：利用泄露的 $d$ 高位恢复 $k$，把 RSA 私钥关系降模到 $e$ 上获得素因子余数，再做已知模余数的因子小根攻击。
- 识别信号：当 $e$ 明显小于 $n$ 且泄露了足够多的 $d$ 高位时，不应只考虑 Boneh–Durfee；先检查 $ed-1=k\varphi(n)$ 是否能确定 $k$ 并泄露 $p+q$ 的模信息。
- 复用要点：模 $e$ 的二次方程有两个根；Coppersmith 输出也只是候选。根分支、枚举高位和最终整除验证缺一不可。
