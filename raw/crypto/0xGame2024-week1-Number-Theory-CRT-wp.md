# Number-Theory-CRT

## 题目简述

题目给出一个约 60 bit 的 RSA 模数、偶数公钥指数和密文：

```text
N = 1022053332886345327
e = 294200073186305890
c = 107033510346108389
```

源码没有强制生成 $\gcd(e,\varphi(N))=1$，本题实际满足 $\gcd(e,\varphi(N))=2$，所以标准 RSA 私钥指数 $e^{-1}\bmod\varphi(N)$ 不存在。flag 定义为 `MD5(str(m))`，目标是恢复随机明文整数 $m$。

## 解题过程

先分解小模数：

```text
p = 970868179
q = 1052721013
```

令 $e_0=e/2$。本题满足 $\gcd(e_0,\varphi(N))=1$，因此可以求

$D\equiv e_0^{-1}\pmod{\varphi(N)}$。

由 $c\equiv m^e=m^{2e_0}\pmod N$ 可得：

$M\equiv c^D\equiv m^2\pmod N$。

问题由 RSA 解密转化为合数模数上的平方根。分别求 $M$ 在模 $p$、模 $q$ 下的两个平方根，再用中国剩余定理组合，会得到 $2\times2=4$ 个候选明文。

下面的脚本使用 SymPy 完成因数分解、模平方根和 CRT，避免原题解中对接近 $10^9$ 的模数逐项暴力枚举：

```python
from hashlib import md5

from sympy import factorint
from sympy.ntheory import sqrt_mod
from sympy.ntheory.modular import crt

N = 1022053332886345327
e = 294200073186305890
c = 107033510346108389

p, q = factorint(N).keys()
phi = (p - 1) * (q - 1)
e0 = e // 2
D = pow(e0, -1, phi)
M = pow(c, D, N)

roots_p = sqrt_mod(M, p, all_roots=True)
roots_q = sqrt_mod(M, q, all_roots=True)

for rp in roots_p:
    for rq in roots_q:
        m = int(crt([p, q], [rp, rq])[0])
        digest = md5(str(m).encode()).hexdigest()
        print(f"0xGame{{{digest}}}")
```

四个候选为：

```text
0xGame{15820afdb9a129e89e40e57f40ff8de9}
0xGame{3932f6728585abbf751a212f69276d3e}
0xGame{f4107420d94cc7037114376d8566d4ef}
0xGame{127016d0be858ef48a99723710ad4d49}
```

题目没有再提供区分 $m$ 与其他平方根的冗余信息，因此逐个验证四个候选即可。

## 方法总结

- 核心技巧：当 $\gcd(e,\varphi(N))=2$ 时，把指数除去公因数，先恢复 $m^2$，再对两个素数模数开平方并用 CRT 合并。
- 识别信号：RSA 指数不可逆但与 $\varphi(N)$ 只有很小公因数，且模数可分解。
- 复用要点：每个素数模数通常产生两个平方根，CRT 会形成多个候选；不要用线性暴力搜索模平方根，应使用 Tonelli–Shanks、Sage 或 `sqrt_mod()`。
