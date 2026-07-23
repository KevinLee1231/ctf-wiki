# Relatively Rough RSA

## 题目简述

题目使用两个数值非常接近的素数生成 RSA 模数，适合用 Fermat 分解。与此同时公开指数是偶数，不能直接在 $\varphi(n)$ 上求逆；需要先恢复一次平方前的值，再处理 Rabin 型四根歧义。

## 解题过程

先从 $\lceil\sqrt n\rceil$ 开始寻找满足 $a^2-n=b^2$ 的整数：

```python
import gmpy2

a = gmpy2.isqrt(n)
if a * a < n:
    a += 1

while True:
    b2 = a * a - n
    b, exact = gmpy2.iroot(b2, 2)
    if exact:
        p = int(a - b)
        q = int(a + b)
        break
    a += 1
```

分解后令 $e'=e/2$。先计算 $d'=(e')^{-1}\bmod\varphi(n)$，得到：

$$
y \equiv c^{d'} \equiv m^2 \pmod n
$$

随后分别在模 $p$、模 $q$ 下求 $y$ 的平方根，并用中国剩余定理组合成四个候选。把候选转换为字节，选择带有赛事前缀的一项：

```text
UMDCTF-{r@bin_crypt0_0v3r_rs@}
```

## 方法总结

本题有两个独立弱点：近素数使 $n$ 可被 Fermat 快速分解，偶数指数又把末步变成 Rabin 平方根问题。不能把 $e$ 当作普通 RSA 指数强行求逆；正确做法是先逆掉 $e/2$，再枚举平方根。
