# BYUCTF 2023 - RSA3

## 题目简述

附件给出两个 RSA 模数和对应密文。两个模数分别形如 $n_1=pq_1$、$n_2=pq_2$，错误地复用了同一个素数 $p$。

## 解题过程

无需分解算法，直接计算最大公因数：

```python
from math import gcd

p = gcd(n1, n2)
q1 = n1 // p
q2 = n2 // p
```

只要 $p\ne 1$，两个私钥都随之恢复。任选与目标密文对应的模数计算：

```python
phi1 = (p - 1) * (q1 - 1)
d1 = pow(e, -1, phi1)
m = pow(c1, d1, n1)
```

字节化后得到：

```text
byuctf{coprime_means_factoring_N_becomes_much_easier}
```

官方说明中“share a common prime (coprime)”措辞自相矛盾；准确说法是两个模数不互素，$\gcd(n_1,n_2)=p$。

## 方法总结

RSA 批量审计应对所有模数做成对或乘积树 GCD。单个模数看起来足够大，但只要不同密钥复用一个素因子，求最大公因数就能同时击穿它们。
