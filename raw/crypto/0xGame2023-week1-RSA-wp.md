# RSA

## 题目简述

RSA 模数 $n$ 不是两个大素数的乘积，而是 16 个较小素数的乘积，因此可以直接分解并计算欧拉函数。RSA 解密后的值仍被一个与 $n$ 不互素的 `mask` 乘过，不能直接求 `mask` 在模 $n$ 下的逆元，需要先提取它们的最大公因数。

## 解题过程

先分解 $n=\prod p_i$。各因子互不相同，所以：

$$
\varphi(n)=\prod_i(p_i-1),\qquad d=e^{-1}\pmod{\varphi(n)}.
$$

计算 $x=c^d\bmod n$，得到 $x\equiv mask\cdot m\pmod n$。令 $g=\gcd(mask,n)$，则 `mask` 本身没有逆元，但 $mask/g$ 与 $n$ 互素。先乘其逆元，再根据本题明文范围除以 $g$，即可恢复 $m$。

```python
from math import prod
from Crypto.Util.number import GCD, inverse, long_to_bytes

n = ...
e = 65537
c = ...
mask = ...

factors = [
    2329990801, 2436711469, 2732757047, 2770441151,
    2821163021, 2864469667, 2995527113, 3111632101,
    3162958289, 3267547559, 3281340371, 3479527847,
    3561068417, 3978177241, 4134768233, 4160088337,
]
assert prod(factors) == n

phi = prod(p - 1 for p in factors)
d = inverse(e, phi)
x = pow(c, d, n)

g = GCD(mask, n)
reduced = x * inverse(mask // g, n) % n
m = reduced // g
print(long_to_bytes(m))
```

模数可用整数分解工具获得，但解题不依赖某个在线站点：得到完整素因子列表后，正文中的公式和脚本即可独立复现全部计算。

## 方法总结

多素数 RSA 的解密框架不变，区别只在于 $\varphi(n)$ 要包含所有素因子。模意义下“除法”必须转换为乘逆元，而逆元存在的前提是两数互素；遇到不可逆乘数时，应先计算最大公因数并判断约分及明文范围条件，而不是直接调用 `inverse()`。
