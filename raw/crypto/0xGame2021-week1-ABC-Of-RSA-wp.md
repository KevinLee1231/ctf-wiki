# week1ABC Of RSA

## 题目简述

已知 RSA 素数 $p=9677$、$q=9241$ 与公钥指数 $e=10009$，要求按题目采用的传统定义计算解密指数 $d$。

## 解题过程

RSA 的欧拉函数为

$$
\varphi(n)=(p-1)(q-1).
$$

解密指数满足 $ed\equiv1\pmod{\varphi(n)}$，因此直接求 $e$ 在模 $\varphi(n)$ 下的逆元。仓库中的判题说明明确要求使用 $\varphi(n)$，所以这里不能把模数换成 $\operatorname{lcm}(p-1,q-1)$，即使后者在一般 RSA 解密中也可能得到可用指数。

```python
from Crypto.Util.number import inverse

p = 9677
q = 9241
e = 10009
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
print(f"0xGame{{{d}}}")
```

输出：

```text
0xGame{39982249}
```

## 方法总结

已知 RSA 的 $p,q,e$ 时，先计算 $\varphi(n)$，再求模逆即可得到 $d$。本题的易错点不是运算，而是必须遵循题目指定的解密指数定义。
