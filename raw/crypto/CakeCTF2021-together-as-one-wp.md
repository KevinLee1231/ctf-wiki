# CakeCTF2021 Together as one

## 题目简述

题目使用三个 512 位强素数 $p,q,r$ 构造 $n=pqr$，并给出：

$$
x=(p+q)^r\bmod n,qquad y=(p+qr)^r\bmod n.
$$

flag 则按普通 RSA 形式加密为 $c=m^{65537}\bmod n$。突破点不在分解通用三素数 RSA，而在 $x,y$ 的差值和它们在不同素因子下的同余性质。

## 解题过程

### 从差值提取 q

模 $q$ 时，两个底数都与 $p$ 同余：

$$
p+q\equiv p+qr\equiv p\pmod q.
$$

因此 $x-y$ 必定被 $q$ 整除，可以直接计算

$$
q=\gcd(n,x-y).
$$

### 利用模 r 的费马关系提取 r

模 $r$ 时，$p+qr\equiv p$。由于 $r$ 是素数，费马小定理给出 $a^r\equiv a\pmod r$，所以

$$
x\equiv p+q\pmod r,qquad y\equiv p\pmod r.
$$

从第一式可得 $p\equiv x-q\pmod r$，于是

$$
r=\gcd\left(\frac nq,,y-(x-q)\right).
$$

最后计算 $p=n/(qr)$，便完成模数分解。

### 标准 RSA 解密

```python
from Crypto.Util.number import GCD, inverse, long_to_bytes

q = GCD(n, x - y)
r = GCD(n // q, y - (x - q))
p = n // (q * r)
assert p * q * r == n

phi = (p - 1) * (q - 1) * (r - 1)
d = inverse(0x10001, phi)
print(long_to_bytes(pow(c, d, n)))
```

对发布附件运行仓库官方脚本，得到：

```text
CakeCTF{This_chall_is_inspired_by_this_music__Check_out!__https://www.youtube.com/watch?v=vLadkYLi8YE_cf49dcb6a31f}
```

flag 内的 URL 是题目内容的一部分，不需要访问它才能完成解题。

## 方法总结

- 辅助值一旦在某个素因子下发生碰撞，`gcd(n, difference)` 就可能直接泄露该因子。
- 对指数恰好等于素数的表达式，应检查费马小定理在对应模数下是否把高次幂降回一次式。
- 多素数 RSA 的安全性仍依赖全部素因子保密；泄露两个因子后，后续就是标准私钥恢复。
