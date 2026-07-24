# RSA 1

## 题目简述

附件提供 RSA 公钥和一段二进制密文。模数没有小因子，也不是完全平方数，但两个素因子距离极近，因此可以用 Fermat 分解。

## 解题过程

Fermat 分解把奇数模数写成平方差：

$$
n=a^2-b^2=(a-b)(a+b)
$$

从 $a=\lceil\sqrt n\rceil$ 开始递增，直到 $a^2-n$ 是完全平方数。本题只需很少迭代便得到两个因子，且：

```text
q - p = 10658
```

完整解密流程如下：

```python
from math import isqrt
from Crypto.PublicKey import RSA
from Crypto.Util.number import inverse, long_to_bytes

key = RSA.import_key(open("public_key", "rb").read())
n, e = key.n, key.e

a = isqrt(n)
if a * a < n:
    a += 1

while True:
    b2 = a * a - n
    b = isqrt(b2)
    if b * b == b2:
        break
    a += 1

p, q = a - b, a + b
assert p * q == n
d = inverse(e, (p - 1) * (q - 1))

c = int.from_bytes(open("encrypted", "rb").read(), "big")
block = long_to_bytes(pow(c, d, n), (n.bit_length() + 7) // 8)
print(block.split(b"\x00", 2)[2].decode())
```

得到：

```text
UMDCTF-{G00D_Th1ng_Peopl3_DonT_Gen3rate_K3ys_Like_This_IN_Pract1ce_Right?}
```

其 SHA-256 与官方摘要 `dacca86d162afd68a741d5072d93e5b9a0718a0cba260dc06a41785a5e9f3ef9` 一致。

## 方法总结

RSA 素数不仅要足够大，还必须独立、随机地产生。若 $p$ 与 $q$ 太接近，$\sqrt n$ 附近就能快速找到平方差，模数位数再大也无济于事。Fermat 分解后仍要按密文块长度保留前导零，才能正确识别 PKCS#1 v1.5 填充。
