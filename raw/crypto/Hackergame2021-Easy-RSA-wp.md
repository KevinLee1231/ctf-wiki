# Easy RSA

## 题目简述

题目给出 RSA 密文以及素数 $p$、$q$ 的特殊生成过程。困难不在分解普通 RSA 模数，而在于逆转两段参数生成：$p$ 来自一个接近素数模数的巨大阶乘，$q$ 则被藏在由十个连续素数组成的多素数 RSA 中。

## 解题过程

### 用 Wilson 定理恢复 p

题目把 $y!\bmod x$ 再取下一素数作为 $p$，其中 $x$ 是素数且 $y$ 很接近 $x$。直接计算 $y!$ 没有必要。Wilson 定理给出：

$$
(x-1)!\equiv-1\pmod{x}
$$

而：

$$
(x-1)!=y!\prod_{i=y+1}^{x-1}i
$$

因此：

$$
y!\equiv-\left(\prod_{i=y+1}^{x-1}i\right)^{-1}\pmod{x}
$$

只需遍历很短的尾部区间：

```python
import gmpy2
import sympy

def recover_p(x, y):
    value = x - 1  # -1 mod x
    for i in range(y + 1, x):
        value = value * gmpy2.invert(i, x) % x
    return sympy.nextprime(int(value))
```

### 用 Euler 定理恢复 q

第二段的 $x$ 位于一组连续素数附近。先向前寻找 9 个素数，再向后取 10 个连续素数 $q_0,\ldots,q_9$，构造：

$$
N=\prod_{i=0}^{9}q_i,\qquad
\varphi(N)=\prod_{i=0}^{9}(q_i-1)
$$

题目给出的 $y$ 是某个值在该多素数 RSA 下的加密结果，因此可直接计算私钥指数并解密：

```python
def recover_q(x, y, e):
    for _ in range(9):
        x = sympy.prevprime(x)

    primes = [x]
    for _ in range(9):
        primes.append(sympy.nextprime(primes[-1]))

    n = 1
    phi = 1
    for prime in primes:
        n *= prime
        phi *= prime - 1

    d = gmpy2.invert(e, phi)
    value = gmpy2.powmod(y, d, n)
    return sympy.nextprime(int(value))
```

### 解开最终密文

得到 $p$、$q$ 后回到标准 RSA：

```python
from Crypto.Util.number import long_to_bytes

n = p * q
d = gmpy2.invert(e, (p - 1) * (q - 1))
m = gmpy2.powmod(c, d, n)
print(long_to_bytes(int(m)))
```

输出即为最终 flag。

## 方法总结

- 核心技巧：用 Wilson 定理把巨大阶乘改写成短尾区间的模逆乘积，再用已知多素数因子计算 Euler 函数和 RSA 私钥。
- 识别信号：阶乘模素数且两个参数非常接近；RSA 模数由一小段可枚举的连续素数构成。
- 复用要点：不要真的展开巨大阶乘；先利用参数生成关系缩短计算。多素数 RSA 的 $\varphi(N)$ 是各互异素因子减一后的乘积。
