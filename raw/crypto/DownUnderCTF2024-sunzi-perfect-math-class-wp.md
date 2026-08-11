# Sun Zi's Perfect Math Class

## 题目简述

题目第二阶段给出同一明文 $m$ 在三组互异 RSA 模数下的密文：

$$
c_i \equiv m^3 \pmod {n_i},\qquad i=1,2,3.
$$

三个 $n_i$ 均为独立生成的 1024 位 RSA 模数，指数固定为 $e=3$，且没有使用随机填充。这里的关键不是分解任一模数，而是三次传输泄露了同一个整数 $m^3$ 在 $n_1,n_2,n_3$ 下的余数；这正是 Håstad 广播攻击的前提。题面前半段的“孙子定理”练习也是对此的提示，因此按密码学而非 Web 前端归类。

## 解题过程

设总模数为 $N=n_1n_2n_3$，并令 $N_i=N/n_i$。由于模数两两互素，可以按中国剩余定理重组：

$$
C=\sum_{i=1}^{3}c_iN_i(N_i^{-1}\bmod n_i)\pmod N.
$$

`src/utils/generate_hastad.py` 先计算的正是这一个 $C$。明文长度远小于三个模数乘积的立方根，所以 $m^3<N$；因此 CRT 给出的不是仅在模 $N$ 下同余的值，而是整数意义上的 $C=m^3$。取精确整数立方根即可还原字节串。

官方 `solve/solve.py` 的核心可简化为：

```python
from functools import reduce
from Crypto.Util.number import inverse, long_to_bytes
import gmpy2

N = n1 * n2 * n3
C = (n2*n3*inverse(n2*n3, n1)*c1
   + n1*n3*inverse(n1*n3, n2)*c2
   + n1*n2*inverse(n1*n2, n3)*c3) % N
m, exact = gmpy2.iroot(C, 3)
assert exact
print(long_to_bytes(m))
```

这里必须检查 `exact`：若三份密文不是同一明文、模数不互素，或 $m^3\ge N$，则不能把近似立方根误当成答案。恢复出的明文为：

```text
DUCTF{btw_y0u_c4n_als0_us3_CRT_f0r_p4rt14l_fr4ct10ns}
```

## 方法总结

低公开指数本身并不必然不安全；漏洞是“同一未填充明文 + 至少 $e$ 个互素模数”。对 $e=3$，三份传输已经足够。实际 RSA 加密应使用 OAEP 等随机填充，并避免把相同裸明文直接广播到多把公钥。这个题也说明 CRT 不仅用于合并普通余数，还可先恢复一个小幂的完整整数值，再开精确整数根。
