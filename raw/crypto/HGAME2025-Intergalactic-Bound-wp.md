# HGAME 2025 Intergalactic Bound

## 题目简述

题目在有限域 $\mathbb F_p$ 上使用 Twisted Hessian 曲线：

$$
a x^3+y^3+1=dxy,
$$

随机选择标量 $k$ 并计算 $Q=kG$，再以 `SHA256(str(k))` 作为 AES-256-ECB 密钥加密 flag。题目公开了 $p$、$G$、$Q$ 和密文，却隐藏曲线参数 $a,d$。先利用 $G,Q$ 都在曲线上这一事实解出 $a,d$，再把三次曲线映射为 Sage 可处理的 Weierstrass 椭圆曲线，求解离散对数 $k$，最后还原 AES 明文。

## 解题过程

### 1. 整理已知量和曲线运算

题目给出：

```python
p = 55099055368053948610276786301
G = (
    19663446762962927633037926740,
    35074412430915656071777015320,
)
Q = (
    26805137673536635825884330180,
    26376833112609309475951186883,
)
ciphertext = (
    b"k\xe8\xbe\x94\x9e\xfc\xe2\x9e\x97\xe5\xf3\x04"
    b"'\x8f\xb2\x01T\x06\x88\x04\xeb3Jl\xdd Pk$\x00:\xf5"
)
```

曲线单位元写作 `(0, 0)`，题目给出的加法为：

$$
(x_1,y_1)+(x_2,y_2)=
\left(
\frac{x_1-y_1^2x_2y_2}{a x_1y_1x_2^2-y_2},
\frac{y_1y_2^2-a x_1^2x_2}{a x_1y_1x_2^2-y_2}
\right).
$$

所有除法均在 $\mathbb F_p$ 中通过模逆完成。标量乘法是普通 double-and-add：

```python
def add_th_curve(P, Q, a, p):
    if P == (0, 0):
        return Q
    if Q == (0, 0):
        return P

    x1, y1 = P
    x2, y2 = Q
    denominator = (a * x1 * y1 * x2**2 - y2) % p
    inv = pow(denominator, -1, p)

    x3 = (x1 - y1**2 * x2 * y2) * inv % p
    y3 = (y1 * y2**2 - a * x1**2 * x2) * inv % p
    return x3, y3

def mul_th_curve(n, P, a, p):
    R = (0, 0)
    while n > 0:
        if n & 1:
            R = add_th_curve(R, P, a, p)
        P = add_th_curve(P, P, a, p)
        n //= 2
    return R
```

### 2. 用两个公开点恢复 $a,d$

任意曲线上点 $(x_i,y_i)$ 都满足：

$$
a x_i^3-dx_iy_i=-(y_i^3+1).
$$

把 $G=(x_G,y_G)$ 和 $Q=(x_Q,y_Q)$ 代入，就得到关于 $a,d$ 的二元一次方程组。可以直接做模线性代数，也可以像官方 WP 一样求 Gröbner 基：

```sage
p = 55099055368053948610276786301
G = (19663446762962927633037926740,
     35074412430915656071777015320)
Q = (26805137673536635825884330180,
     26376833112609309475951186883)

R.<a, d> = PolynomialRing(GF(p))
f1 = a*G[0]^3 + G[1]^3 + 1 - d*G[0]*G[1]
f2 = a*Q[0]^3 + Q[1]^3 + 1 - d*Q[0]*Q[1]

I = ideal(f1, f2)
print(I.groebner_basis())
```

输出：

```text
[a + 16017244634673333349551751112,
 d + 46529564890039836205593471940]
```

所以在模 $p$ 意义下：

```text
a = 39081810733380615260725035189
d =  8569490478014112404683314361
```

也可以不用 Gröbner 基，直接由二元方程消元：

```python
xg, yg = G
xq, yq = Q

num = xg*yg*(yq**3 + 1) - xq*yq*(yg**3 + 1)
den = xg*xq*(xg**2*yq - xq**2*yg)
a = num * pow(den, -1, p) % p
d = (a*xg**3 + yg**3 + 1) * pow(xg*yg, -1, p) % p
```

将参数分别代回 $G,Q$ 的曲线方程，余数均为 0。

### 3. 映射到 Weierstrass 曲线

将仿射方程齐次化：

$$
a x^3+y^3+z^3=dxyz.
$$

Sage 可以直接从光滑三次曲线构造椭圆曲线及双有理映射：

```sage
F = GF(p)
R.<x, y, z> = PolynomialRing(F)
cubic = a*x^3 + y^3 + z^3 - d*x*y*z

E = EllipticCurve_from_cubic(cubic, morphism=True)
EG = E(G)
EQ = E(Q)
```

官方 PDF 还给出了一组显式变换。令：

```sage
F = GF(p)

def th_to_weierstrass(P):
    x, y = map(F, P)
    common = d*x/3 + y + 1
    u = (-3/(a - (d/3)^3) * x) / common
    v = (-9/(a - (d/3)^3)^2 * (-y)) / common
    return u, v

a1 = -d / (a - (d/3)^3)
a2 = -9*(d/3)^2 / (a - (d/3)^3)^2
a3 = -9 / (a - (d/3)^3)^2
a4 = -27*(d/3) / (a - (d/3)^3)^3
a6 = -27 / (a - (d/3)^3)^4

E = EllipticCurve(F, [a1, a2, a3, a4, a6])
uG, vG = th_to_weierstrass(G)
uQ, vQ = th_to_weierstrass(Q)
EG = E(uG, vG)
EQ = E(uQ, vQ)
```

两种写法目的相同：把原曲线群上的 $Q=kG$ 保持为 Weierstrass 模型上的 $E_Q=kE_G$，从而调用成熟的椭圆曲线离散对数实现。

### 4. 求离散对数

在 Sage 中可直接求：

```sage
k = EQ.log(EG)
print(k)
```

得到：

```text
2633177798829352921583206736
```

若希望显式控制算法，可先分解 `EG.order()`，对每个素数幂子群求离散对数，再用中国剩余定理组合，即 Pohlig–Hellman：

```sage
def pohlig_hellman(n, G, Q):
    prime_powers = [q^e for q, e in factor(n)]
    residues = []

    for modulus in prime_powers:
        cofactor = n // modulus
        residue = discrete_log(
            cofactor * Q,
            cofactor * G,
            ord=modulus,
            operation="+",
        )
        residues.append(residue)

    return crt(residues, prime_powers)

k = pohlig_hellman(EG.order(), EG, EQ)
```

把 $k$ 代回题目原始 `mul_th_curve`，可验证：

```python
assert mul_th_curve(k, G, a, p) == Q
```

该断言已经对题目参数实际核验通过。

### 5. 派生 AES 密钥并解密

题目把标量的十进制字符串做 SHA-256，得到 32 字节 AES 密钥：

```python
import hashlib
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

k = 2633177798829352921583206736
key = hashlib.sha256(str(k).encode()).digest()

cipher = AES.new(key, AES.MODE_ECB)
plaintext = unpad(cipher.decrypt(ciphertext), 16)
print(plaintext)
```

密文长度为 32 字节，解密后的末尾是四个 `0x04`，符合 PKCS#7 填充。去除填充后得到：

```text
hgame{N0th1ng_bu7_up_Up_UP!}
```

## 方法总结

本题的链条为“用两个公开点恢复曲线参数 → 将 Twisted Hessian 三次曲线转换为 Weierstrass 模型 → 求 $Q=kG$ 的离散对数 → 用 `SHA256(str(k))` 解 AES”。隐藏 $a,d$ 并没有隐藏曲线，因为两个公开点已经给出了两条关于参数的独立线性约束。

验证时应分三层进行：检查 $G,Q$ 都满足恢复后的曲线方程；检查原始加法公式下 $kG=Q$；最后检查 AES 明文的 PKCS#7 填充与 flag 格式。这样可以把“映射方向错”“离散对数基点写反”和“密钥字符串格式错”分别定位。
