# hidden-poly

## 题目简述

题目随机生成 16 字节 AES 密钥，并把每个字节作为多项式系数：

$$
f(x)=\sum_{i=0}^{15}a_ix^i.
$$

已知模数 $q$、根 $r$，且源码保证

$$
f(r)\equiv0\pmod q,
\qquad 0\le a_i<128.
$$

`bytes.isascii()` 只保证每个字节小于 `0x80`，并不保证它们全是可打印字符。目标是从一个模线性关系中恢复 16 个较小系数，再用它们作为 AES-128-CBC 密钥解密 flag。

## 解题过程

已知关系可以写成

$$
\sum_{i=0}^{15}a_i(r^i\bmod q)-tq=0
$$

其中 $t$ 是某个整数。构造 17 维整数格，基向量为

$$
(Kq,0,\ldots,0)
$$

以及对每个 $0\le i<16$ 的

$$
(K(r^i\bmod q),0,\ldots,1_i,\ldots,0).
$$

对应的格基矩阵为

$$
M=
\begin{pmatrix}
Kq&0&0&\cdots&0\\
K&1&0&\cdots&0\\
K(r\bmod q)&0&1&\cdots&0\\
\vdots&\vdots&\vdots&\ddots&\vdots\\
K(r^{15}\bmod q)&0&0&\cdots&1
\end{pmatrix}.
$$

对第一行取系数 $-t$、其余 16 行分别取系数 $a_i$，格中就存在

$$
(0,a_0,a_1,\ldots,a_{15}).
$$

后 16 个分量都小于 128，而任何未消掉的第一分量至少会受到巨大缩放因子 $K=2^{1024}$ 的惩罚。因此该向量相对很短，LLL 可以把它暴露出来。

LLL 只保证给出一组较短基，并不保证目标一定是第一行或符号固定。稳妥做法是遍历约化后的行及其相反数，筛选第一坐标为零、其余坐标均落在 `[0, 128)` 的候选，再用 AES 填充和 flag 前缀确认。

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from sage.all import ZZ, matrix, power_mod

q = 264273181570520944116363476632762225021
root = 122536272320154909907460423807891938232
iv = b'Gc\xf2\xfd\x94\xdc\xc8\xbb\xf4\x84\xb1\xfd\x96\xcd6\\'
ciphertext = bytes.fromhex(
    "d23eac665cdb57a8ae7764bb4497eb2f"
    "79729537e596600ded7a068c407e67ea"
    "75e6d76eb9e23e21634b84a96424130e"
)

K = 1 << 1024
M = matrix(ZZ, 17, 17)
M[0, 0] = K * q
for i in range(16):
    M[i + 1, 0] = K * power_mod(root, i, q)
    M[i + 1, i + 1] = 1

for row in M.LLL().rows():
    if row[0] != 0:
        continue
    for sign in (1, -1):
        coefficients = [int(sign * row[i]) for i in range(1, 17)]
        if not all(0 <= value < 128 for value in coefficients):
            continue
        key = bytes(coefficients)
        try:
            plain = unpad(
                AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext), 16
            )
        except ValueError:
            continue
        if plain.startswith(b"moectf{"):
            print(key)
            print(plain.decode())
```

得到：

```text
moectf{th3_first_blood_0f_LLL!@#$}
```

## 方法总结

本题把“模 $q$ 为零”和“系数很小”组合成了典型的格短向量问题。构造时用巨大 $K$ 放大同余残差，让满足同余的系数向量具有零首坐标和很小的欧氏范数；LLL 负责找到短候选，AES 合法填充与 flag 前缀负责做最终判定。实现中不要把 `isascii()` 误解为“可打印字符”，也不要依赖 `LLL()[0]` 的偶然排序。
