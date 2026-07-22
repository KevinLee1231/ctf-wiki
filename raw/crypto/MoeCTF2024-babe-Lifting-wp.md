# babe-Lifting

## 题目简述

题目泄漏 1024 位 RSA 私钥指数 $d$ 的低 400 位，且公钥指数很小：$e=4097$。先枚举 $ed-1=k\varphi(n)$ 中的 $k$，在模 $2^{400}$ 下恢复 $p$ 的大量低位，再用单变量 Coppersmith 小根攻击补齐 $p$ 的高位并分解 $n$。

## 解题过程

因为 $0<d<\varphi(n)$，有：

$$
ed-1=k\varphi(n),\qquad 1\le k<e.
$$

又有 $\varphi(n)=n+1-(p+q)$。令已知低位为 $d_0=d\bmod2^{400}$，模 $2^{400}$ 可得：

$$
ed_0-1-k(n+1)+k(p+q)\equiv0\pmod{2^{400}}.
$$

由于 $pq=n$ 且奇素数 $p$ 在模 $2^{400}$ 下可逆，两边乘 $p$，把 $q$ 换成 $n/p$：

$$
kp^2+\bigl(ed_0-1-k(n+1)\bigr)p+kn
\equiv0\pmod{2^{400}}.
$$

对每个 $k$ 解这个模方程，可以得到 $p$ 或 $q$ 的低位候选。实现中保守使用其中低 398 位，写成：

$$
p=p_0+2^{398}x,\qquad |x|<2^{114}.
$$

于是多项式 $f(x)=p_0+2^{398}x$ 在模 $n$ 下与未知因子 $p$ 共享大因子；Coppersmith `small_roots` 可找出小根 $x$，再由 $\gcd(f(x),n)$ 取回 $p$。

```python
from Crypto.Util.number import long_to_bytes
from sage.all import GF, GCD, PolynomialRing, ZZ, Zmod, inverse_mod, solve_mod, var

n = 53282434320648520638797489235916411774754088938038649364676595382708882567582074768467750091758871986943425295325684397148357683679972957390367050797096129400800737430005406586421368399203345142990796139798355888856700153024507788780229752591276439736039630358687617540130010809829171308760432760545372777123
e = 4097
c = 14615370570055065930014711673507863471799103656443111041437374352195976523098242549568514149286911564703856030770733394303895224311305717058669800588144055600432004216871763513804811217695900972286301248213735105234803253084265599843829792871483051020532819945635641611821829176170902766901550045863639612054
d0 = 1550452349150409256147460237724995145109078733341405037037945312861833198753379389784394833566301246926188176937280242129

MOD_BITS = 400
KNOWN_BITS = 398

def recover_factor(low):
    ring = PolynomialRing(Zmod(n), "x")
    x = ring.gen()
    polynomial = (2**KNOWN_BITS * x + low).monic()
    roots = polynomial.small_roots(
        X=2 ** (512 - KNOWN_BITS),
        beta=0.49,
    )
    for root in roots:
        factor = GCD(2**KNOWN_BITS * ZZ(root) + low, n)
        if 1 < factor < n:
            return ZZ(factor)
    return None

X = var("X")
p = None
for k in range(1, e):
    equation = (
        k * X**2
        + (e * d0 - 1 - k * (n + 1)) * X
        + k * n
        == 0
    )
    for root in solve_mod([equation], 2**MOD_BITS):
        low = ZZ(root[0]) % (2**KNOWN_BITS)
        p = recover_factor(low)
        if p is not None:
            break
    if p is not None:
        break

assert p is not None and n % p == 0
q = n // p
d = inverse_mod(e, (p - 1) * (q - 1))
print(long_to_bytes(pow(c, int(d), n)))
```

输出：

```text
moectf{7h3_st4rt_0f_c0pp3rsmith!}
```

## 方法总结

低位泄漏先通过 $ed-1=k\varphi(n)$ 转成素因子低位，再把未知高位视为小根。Coppersmith 的作用和参数界已在正文中给出，原稿引用的通用个人博客不再是复现所必需，因此删除该外链。
