# PolyRSA

## 题目简述

题目把 flag 每 8 字节转换为一个整数系数，组成次数小于 8 的多项式 $f$，并在商环

$$
R=(\mathbb Z/n\mathbb Z)[x]/(x^8-1),\qquad n=pq
$$

中计算 $c=f^e$。题目直接给出 $p,q,e,c$，目标是构造适用于该有限环单位群的指数，求逆指数并恢复多项式系数。

## 解题过程

模 $p$ 与模 $q$ 分别考虑时，$x^8-1$ 的不可约因子次数都整除 8；对应有限域分量的非零元素阶分别整除 $p^8-1$ 和 $q^8-1$。因此可取：

$$
\lambda=(p^8-1)(q^8-1)
$$

作为单位群指数的公倍数，并令 $d=e^{-1}\bmod\lambda$。需要注意，$lambda$ 不一定是这个商环“精确的欧拉函数”，但对本题选取的可逆元素足以保证 $f^{ed}=f$。

```sage
from Crypto.Util.number import long_to_bytes

p = ...
q = ...
e = 65537
c_text = "..."  # 复制题目输出的完整多项式

R0.<t> = PolynomialRing(Zmod(p * q))
R.<x> = R0.quotient(t^8 - 1)
c = R(c_text)

group_exponent = (p^8 - 1) * (q^8 - 1)
d = inverse_mod(e, group_exponent)
f = (c^d).lift()

chunks = [long_to_bytes(int(value)) for value in f.list()]
print(b''.join(chunks))
```

`f.list()` 按 $x^0,x^1,\ldots$ 顺序返回系数，正好对应加密时从前到后切出的 8 字节块。若题目允许块以零字节开头，还需额外记录每块长度，避免 `long_to_bytes()` 丢失前导零；本题的 flag 分块不触发该问题。

## 方法总结

- 核心技巧：把 RSA 的“指数求逆”推广到有限商环，并选择单位群指数的可用公倍数。
- 识别信号：消息被编码为多项式、运算发生在 $\mathbb Z_n[x]/(g(x))$、且 $p,q$ 已知。
- 复用要点：不要把 $(p^8-1)(q^8-1)$ 无条件称为精确欧拉函数；应说明它为何是本题单位群阶的公倍数，并确认明文元素可逆及系数顺序。
