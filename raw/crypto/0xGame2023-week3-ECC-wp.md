# ECC

## 题目简述

题目实现了类似椭圆曲线 ElGamal 的加密。私钥为 64 位整数 $k$，公钥 $K=kG$；加密时生成 16 位随机数 $r$，并输出

$$
C_1=M+rK,\qquad C_2=rG.
$$

只要从 $K=kG$ 解出离散对数 $k$，就能计算

$$
M=C_1-kC_2.
$$

题目的额外缺陷是没有把明文规范编码为曲线点，而是直接令 $M=(m_1,m_2)$。因此 $M$ 和 $C_1$ 不一定在曲线上，不能把所有数据都交给 Sage 的曲线点类型处理。

## 解题过程

`G`、`K` 和 `C_2` 是按曲线群运算产生的合法点。先在 Sage 中建立

$$
E: y^2=x^3+ax+b\pmod q
$$

并求 $K$ 关于 $G$ 的离散对数。随后对 `C_2` 做标量乘法，再沿用题目源码中的坐标公式，从 `C_1` 中减去该点。完整脚本如下：

```python
# SageMath
from Crypto.Util.number import inverse, long_to_bytes

q = 1139075593950729137191297
a = 930515656721155210883162
b = 631258792856205568553568

G = (641322496020493855620384, 437819621961768591577606)
K = (781988559490437792081406, 76709224526706154630278)
C1 = (926699948475842085692652, 598672291835744938490461)
C2 = (919875062299924962858230, 227616977409545353942469)

E = EllipticCurve(GF(q), [0, 0, 0, a, b])
curve_G = E.point(G)
curve_K = E.point(K)
k = ZZ(curve_G.discrete_log(curve_K))
print(f"k = {k}")

def point_add(P, Q):
    if P is None:
        return Q
    if Q is None:
        return P
    if P[0] == Q[0] and (P[1] + Q[1]) % q == 0:
        return None

    if P == Q:
        slope = (3 * P[0] * P[0] + a) * inverse(2 * P[1], q)
    else:
        slope = (Q[1] - P[1]) * inverse(Q[0] - P[0], q)
    slope %= q

    x3 = (slope * slope - P[0] - Q[0]) % q
    y3 = (slope * (P[0] - x3) - P[1]) % q
    return x3, y3

def scalar_mul(n, P):
    result = None
    while n:
        if n & 1:
            result = point_add(result, P)
        P = point_add(P, P)
        n >>= 1
    return result

kC2 = scalar_mul(k, C2)
minus_kC2 = (kC2[0], (-kC2[1]) % q)
M = point_add(C1, minus_kC2)

msg = long_to_bytes(M[0]) + long_to_bytes(M[1])
print(msg)
print(b"0xGame{" + msg + b"}")
```

脚本得到私钥

```text
12515237792257199894
```

并恢复消息 `Al1ce_L0ve_B0b`，加上题目源码中的 flag 包装即可。

## 方法总结

本题由“曲线离散对数”和“不规范明文编码”两部分组成。Sage 只用于合法曲线点之间的 $K=kG$；对不在曲线上的 `C1`，必须复用题目实际使用的坐标运算，不能强行调用 `E.point(C1)`。审计自制 ECC 时还应重点检查私钥位数、随机数位数以及明文到曲线的编码方式。
