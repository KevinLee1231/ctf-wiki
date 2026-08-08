# miniLCTF 2024 Curvesignin Revenge Writeup

## 题目简述

题目在模素数 $N$ 上定义了曲线

$$x^2+e y^2\equiv 1\pmod N$$

以及运算

$$
(x_1,y_1)\circ(x_2,y_2)
=(x_1x_2-ey_1y_2,\ x_1y_2+x_2y_1).
$$

Alice 与 Bob 在这个群上执行类似 Diffie-Hellman 的密钥交换，最终取共享点的横坐标，经 SHA-256 截取 16 字节作为 AES-CBC 密钥。攻击目标是由公开点恢复 Alice 的私钥，进而解密密文。

## 解题过程

### 将曲线群映射到有限域乘法群

因为 $e$ 是模 $N$ 的二次剩余，可求 $d^2=e$。在 $\mathbb F_{N^2}=\mathbb F_N[i]/(i^2+1)$ 中作映射

$$\phi(x,y)=x+i d y.$$

曲线方程给出

$$\phi(x,y)\phi(x,-y)=x^2+ey^2=1,$$

而题目定义的点运算恰好对应域元素乘法。因此，曲线上的离散对数可以直接转成 $\mathbb F_{N^2}^{\*}$ 中的离散对数。

对曲线上元素 $s$，Frobenius 自同态给出 $s^N=s^{-1}$，所以 $s^{N+1}=1$。本题中

```text
N + 1 = 2^2 * 7 * 877 * 2269 * 37967 * 184279 * 504877
        * 845833 * 12308089 * 25153483 * 135503999
        * 149848639 * 223321729 * 264522527
```

完全光滑，Sage 的 `discrete_log` 会使用 Pohlig-Hellman 很快求出指数。

### 恢复共享密钥

核心求解代码如下；`G`、`Alice_pub`、`Bob_pub`、`iv` 与 `ct` 取自题目输出。

```python
from sage.all import GF, PolynomialRing, discrete_log, Mod, sqrt
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad

N = 61820395509869592945047899644070363303060865412602815892951881829112472104091
e = 133422

F = GF(N)
R = PolynomialRing(F, "z")
z = R.gen()
K = GF(N**2, name="i", modulus=z**2 + 1)
i = K.gen()
d = F(sqrt(Mod(e, N)))

def lift_point(P):
    x, y = P
    return K(x) + i * K(d * y)

g = lift_point(G)
a_pub = lift_point(Alice_pub)
b_pub = lift_point(Bob_pub)
n_a = discrete_log(a_pub, g, ord=N + 1)

shared_elem = b_pub**n_a
shared_x = int((shared_elem + shared_elem**-1) / 2)
key = sha256(long_to_bytes(shared_x)).digest()[:16]
flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(flag)
```

本地复现得到 Alice 指数

```text
47885227719229463949869731721047296800235918311075465323872183842041075747873
```

共享点横坐标为

```text
55091084772726390359981214393707445864337228035494609797905461889421520721254
```

AES 密钥为 `0bd70807e49a0b8906fc25cc587e8fbb`，解得：

```text
miniLCTF{C0nics_ar3_n0t_elliptic_Curv3s!}
```

## 方法总结

题目把乘法群包装成了二次曲线上的自定义运算。识别同态 $x+idy$ 后，难点就变回有限域离散对数；再利用 $N+1$ 光滑这一条件运行 Pohlig-Hellman。面对自定义“曲线 DH”，应先检查运算是否来自二次扩域中的乘法，而不是直接套椭圆曲线算法。
