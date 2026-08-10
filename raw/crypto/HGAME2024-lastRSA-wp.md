# lastRSA

## 题目简述

题目先生成两个 512-bit strong prime $p,q$，令 $n=pq$，再泄露：

$$
L=p\oplus(q\mathbin{\texttt{>>}}13).
$$

但 $L$ 没有直接输出，而是分别代入两个 40 次幂和，得到 `enc1` 与 `enc2`。恢复 $L$ 后，还要利用异或关系和 $pq=n$ 重建两个素因子，最后完成常规 RSA 解密。

题目生成逻辑为：

```python
from Crypto.Util.number import *
from secret import flag


def encrypt(modulus, selector, leak0):
    rounds = 40
    t = 114514
    x = leak0 + 2 * t if selector == 1 else 2 * t * leak0
    result = 2024
    while rounds:
        result += pow(x, rounds, modulus)
        rounds -= 1
    return result


message = bytes_to_long(flag)
p = getStrongPrime(512)
q = getStrongPrime(512)
e = 0x10001
leak0 = p ^ (q >> 13)
n = p * q
enc1 = encrypt(n, 1, leak0)
enc2 = encrypt(n, 0, leak0)
ciphertext = pow(message, e, n)
```

## 解题过程

### 把两组泄漏写成同根多项式

记 $t=114514$、未知量为 $L$。虽然 `encrypt` 把每一项分别模 $n$ 后相加，没有对总和再次取模，但仍有同余关系：

$$
f_1(L)=2024-\texttt{enc1}
+\sum_{i=1}^{40}(L+2t)^i\equiv0\pmod n,
$$

$$
f_2(L)=2024-\texttt{enc2}
+\sum_{i=1}^{40}(2tL)^i\equiv0\pmod n.
$$

两个多项式在 $\mathbb Z_n$ 上有公共根 $L$。可以对它们求多项式 GCD；官方 PDF 给出的更短做法是在商环上求 Gröbner 基，结果中会出现关于 $L$ 的一次多项式：

```sage
t = 114514
R.<L> = PolynomialRing(Zmod(n))

f1 = R(2024 - enc1)
f2 = R(2024 - enc2)
for exponent in range(1, 41):
    f1 += (L + 2 * t)**exponent
    f2 += (2 * t * L)**exponent

basis = Ideal([f1, f2]).groebner_basis()
linear = next(poly for poly in basis if poly.degree() == 1)
leak0 = int(-linear[0] / linear[1]) % n
print(leak0)
```

这里不应机械假设基的第一个元素或首一多项式的常数项符号；显式筛选一次式并按 $a_1L+a_0=0$ 计算 $L=-a_0a_1^{-1}$ 更稳妥。恢复结果为：

```text
13168452015078389807681744077701012683188749953280204324570483361963541298704796389757190180549802771265899020301416729606658667351017116721327316272373584
```

把该值重新代入两组 40 次幂和并与 `enc1 % n`、`enc2 % n` 比较，可以确认公共根没有取错。

### 利用移位异或恢复因子

关系 $L=p\oplus(q\mathbin{\texttt{>>}}13)$ 有两个重要性质：

- $q\mathbin{\texttt{>>}}13$ 最多只有 499 bit，因此 $L$ 的最高 13 bit 就是 $p$ 的最高 13 bit；
- 从最低位看，若已知 $q$ 的低 13 bit，则每扩展一位 $p_i$，都可由 $L_i=p_i\oplus q_{i+13}$ 得到对应的 $q_{i+13}$，再用 $pq\equiv n\pmod{2^k}$ 立即剪枝。

官方 PDF 使用第二种逐位搜索。初始只需枚举约 $2^{13}$ 种 $q$ 低位；每一层按乘积的低位同余过滤，因此总工作量约为 $512\times2^{13}$ 量级，而不是暴力枚举两个 512-bit 素数。

还可以从最高位交替逼近，利用 $n/p$ 估计 $q$ 的高位，再通过异或关系扩展 $p$ 的已知前缀。下面这份经过正向验证的脚本采用后一种方法；末尾只剩很少低位需要枚举：

```python
from Crypto.Util.number import inverse, long_to_bytes


enc1 = 2481998981478152169164378674194911111475668734496914731682204172873045273889232856266140236518231314247189371709204253066552650323964534117750428068488816244218804456399611481184330258906749484831445348350172666468738790766815099309565494384945826796034182837505953580660530809234341340618365003203562639721024
enc2 = 2892413486487317168909532087203213279451225676278514499452279887449096190436834627119161155437012153025493797437822039637248773941097619806471091066094500182219982742574131816371999183859939231601667171386686480639682179794271743863617494759526428080527698539121555583797116049103918578087014860597240690299394
ciphertext = 87077759878060225287052106938097622158896106278756852778571684429767457761148474369973882278847307769690207029595557915248044823659812747567906459417733553420521047767697402135115530660537769991893832879721828034794560921646691417429690920199537846426396918932533649132260605985848584545112232670451169040592
n = 136159501395608246592433283541763642196295827652290287729738751327141687762873360488671062583851846628664067117347340297084457474032286451582225574885517757497232577841944028986878525656103449482492190400477852995620473233002547925192690737520592206832895895025277841872025718478827192193010765543046480481871
leak0 = 13168452015078389807681744077701012683188749953280204324570483361963541298704796389757190180549802771265899020301416729606658667351017116721327316272373584


def leak_polynomial(modulus, selector, value):
    t = 114514
    x = value + 2 * t if selector == 1 else 2 * t * value
    return (
        2024
        + sum(pow(x, exponent, modulus) for exponent in range(1, 41))
    ) % modulus


# q >> 13 不影响 p 所在的最高 13 bit。
p_prefix = leak0 >> (512 - 13)

while True:
    # 用当前 p 前缀和 n 的高位估算 q 前缀。
    q_prefix = (
        n >> (1024 - 2 * p_prefix.bit_length())
    ) // p_prefix
    q_prefix >>= 4

    shift = 512 - 13 - q_prefix.bit_length()
    if shift < 0:
        break

    recovered = leak0 ^ (q_prefix << shift)
    new_prefix = recovered >> shift
    if new_prefix == p_prefix:
        break
    p_prefix = new_prefix

# 此时只剩极少低位；多枚举两位可容纳逼近误差。
p = None
for low_bits in range(64):
    candidate = (p_prefix << 4) + low_bits
    if candidate > 1 and n % candidate == 0:
        p = candidate
        break

if p is None:
    raise RuntimeError("factor recovery failed")

q = n // p

# 三项正向验证，避免把近似前缀误当成真实因子。
assert p * q == n
assert p ^ (q >> 13) == leak0
assert leak_polynomial(n, 1, leak0) == enc1 % n
assert leak_polynomial(n, 0, leak0) == enc2 % n

e = 0x10001
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
message = pow(ciphertext, d, n)
assert pow(message, e, n) == ciphertext
print(long_to_bytes(message))
```

恢复出的两个素因子为：

```text
p = 13167244882304693277785720567493996610066918256369682594482416913362069704726831109204371100970154866396462315730687841430922916219416627940866383413192931
q = 10340773837858169661474323029012384377394391882332560606952494899463284596209932089793576041492039641919331765221984085549386070977506894068717765568920741
```

### RSA 解密与结果验证

计算：

$$
\varphi(n)=(p-1)(q-1),\qquad
d=e^{-1}\bmod\varphi(n),
$$

再求 $m=c^d\bmod n$ 并转为字节，得到：

```text
hgame{Gr0bn3r_ba3ic_0ften_w0rk3_w0nd3rs}
```

本地复算同时验证了 $pq=n$、$p\oplus(q\gg13)=L$、两组泄漏多项式和 `pow(m, e, n) == ciphertext`。[出题人赛后公开的题目源码与补充解法](https://pythok.icu/2024/03/20/2024-Hgame-week4-wp-crypto/)提供了官方 PDF 未包含的原始 `encrypt` 定义；关键等式、恢复值和验证步骤均已写入正文。

## 方法总结

- 自定义泄漏函数只要能写成未知量上的多项式，就应先检查多个输出是否共享同一根；本题可用 Gröbner 基或多项式 GCD 把 40 次关系直接降为一次式。
- `pow(x, i, n)` 逐项取模后再求和，不影响最终的模 $n$ 多项式关系，但比较时应使用 `enc % n`。
- $p\oplus(q\gg s)$ 同时泄露位对齐关系和一段未受影响的高位。结合 $pq=n$ 的逐位同余，可以把指数级空间压缩到约 $2^s$ 个初始状态。
- 前缀逼近算法速度很快，但属于带工程假设的恢复方法；候选因子必须用整除、异或泄漏和原始泄漏函数逐项复核。
- 得到 $p,q$ 后仍应做 RSA 正向回验，确认解出的明文重新加密确实等于给定密文。
