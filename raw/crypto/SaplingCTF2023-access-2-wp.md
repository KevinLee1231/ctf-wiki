# access-2

## 题目简述

这次 AES 密钥来自 4 个随机字节，但被拆成两个独立的 2 字节值，每个值重复扩展为一个 16 字节 AES 密钥，并连续加密两次。直接枚举全部组合需要 $2^{32}$ 次双重加密；已知一组明文和密文后，可以用中间相遇把复杂度降到约 $2^{17}$ 次 AES 运算和 $2^{16}$ 级内存。

## 解题过程

设两层密钥为 $K_1$、$K_2$，已知：

$$
C=E_{K_2}(E_{K_1}(P))
$$

先枚举所有 $K_1$，保存中间值 $E_{K_1}(P)$；再枚举 $K_2$，计算 $D_{K_2}(C)$ 并查表：

~~~python
from Crypto.Cipher import AES

def expand(x):
    return x.to_bytes(2, "big") * 8

middle = {}
for x in range(1 << 16):
    v = AES.new(expand(x), AES.MODE_ECB).encrypt(known_plaintext)
    middle[v] = x

for y in range(1 << 16):
    v = AES.new(expand(y), AES.MODE_ECB).decrypt(known_ciphertext)
    if v in middle:
        k1 = expand(middle[v])
        k2 = expand(y)
        break
~~~

再按相同顺序解开目标密文，并检查填充和 flag 前缀，得到：

~~~text
maple{d1V1d3_4ND_c0nQW3R!}
~~~

## 方法总结

多重加密不会自动把有效安全强度相乘。只要存在可验证的中间状态，双重加密就可能受到 meet-in-the-middle 攻击。本题还把每层密钥限制为 16 位，使前向表和反向枚举都很小；写脚本时要特别确认密钥字节序、重复方式以及两层加解密的先后顺序。
