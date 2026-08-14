# Intro to RSA

## 题目简述

题目直接给出 RSA 的两个 512 位素数 $p,q$、公钥指数 $e=65537$ 和密文 $c$，要求计算私钥指数 $d$，再恢复消息整数 $m$。这是 RSA 私钥计算与模幂解密的入门题。

## 解题过程

先计算模数 $N=pq$。私钥指数满足：

$$ed\equiv1\pmod{\varphi(N)},\qquad \varphi(N)=(p-1)(q-1)$$

因此用模逆得到 $d=e^{-1}\bmod\varphi(N)$，再计算 $m=c^d\bmod N$。整数消息最后按大端字节串转换：

```python
from Crypto.Util.number import long_to_bytes

N = p * q
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, N)
print(long_to_bytes(m))
```

服务端实际接受满足 $ed\equiv1\pmod{\operatorname{lcm}(p-1,q-1)}$ 的 $d$，使用上式按 $\varphi(N)$ 求逆同样满足要求。转换结果为：

```text
grey{c0ngr4tz_u_c4n_d0_RSA_04728642737968273931467983}
```

## 方法总结

已知 $p$、$q$ 时 RSA 不再依赖分解难题：可直接求出 $\varphi(N)$ 或 Carmichael 函数，进而计算私钥并解密。实现时要注意 RSA 运算对象是整数，字符串与整数之间需显式转换。
