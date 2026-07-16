# What's CRT

## 题目简述

题目给出两组素数和与乘积：$p+q$、$pq=n$ 以及 $p'+q'$、$p'q'=n'$，可直接解二次方程得到四个素因子。RSA 公钥指数与 $\varphi(n)$ 的最大公因数为 4，无法按普通 RSA 求 $e^{-1}$；但可先恢复 $m^4$ 在四个素数模数下的余数，再用中国剩余定理重建精确的 $m^4$ 并开四次整数根。

## 解题过程

### 由和与积恢复素数

$p$、$q$ 是方程 $x^2-Sx+N=0$ 的两根，其中 $S=p+q$、$N=pq$：

$$
p,q=\frac{S\pm\sqrt{S^2-4N}}{2}.
$$

```python
from math import isqrt

def factor_from_sum(n, total):
    delta2 = total * total - 4 * n
    delta = isqrt(delta2)
    assert delta * delta == delta2
    p = (total + delta) // 2
    q = (total - delta) // 2
    assert p * q == n
    return p, q

p, q = factor_from_sum(n, mygift[0])
p_, q_ = factor_from_sum(n_, mygift[1])
```

### 把 RSA 降为四次幂余数

令 $e=4e'$。本题满足 $\gcd(e,\varphi(n))=4$ 且 $\gcd(e',\varphi(n))=1$，所以可求 $d'=(e')^{-1}\bmod\varphi(n)$：

$$
c^{d'}\equiv m^{ed'}=m^{4e'd'}\equiv m^4\pmod n.
$$

```python
from Crypto.Util.number import GCD, inverse

phi = (p - 1) * (q - 1)
assert GCD(e, phi) == 4
d4 = inverse(e // 4, phi)
m4_mod_n = pow(c, d4, n)
rp = m4_mod_n % p
rq = m4_mod_n % q
```

题目另外直接给出 `mp_ = m^4 mod p_` 和 `mq_ = m^4 mod q_`。四个素数两两互素，对余数 $a_i$ 和模数 $m_i$，令 $M=\prod m_i$、$M_i=M/m_i$，CRT 解为：

$$
x\equiv\sum_i a_iM_i(M_i^{-1}\bmod m_i)\pmod M.
$$

```python
from math import prod
from gmpy2 import iroot
from Crypto.Util.number import long_to_bytes

def crt(residues, moduli):
    M = prod(moduli)
    x = 0
    for a, mod in zip(residues, moduli):
        Mi = M // mod
        x = (x + a * Mi * inverse(Mi, mod)) % M
    return x

m4 = crt(
    [rp, rq, mp_, mq_],
    [p, q, p_, q_],
)
root, exact = iroot(m4, 4)
assert exact
print(long_to_bytes(int(root)))
```

四个 512 位素数的乘积远大于本题实际的 $m^4$，所以 CRT 在该范围内恢复的不是仅有同余意义的代表元，而是精确整数 $m^4$；这也是最后可以直接开整数根的必要条件。

## 方法总结

当 $\gcd(e,\varphi(n))>1$ 时，普通 RSA 私钥逆元不存在，应先判断能否恢复 $m^g$ 的各模余数。CRT 只能在模数两两互素时唯一合并，并且要额外证明目标整数小于总模数，才能把同余结果当作精确幂值开根；原稿缺失的证明图片已由正文公式完整替代。
