# ECC-baby

## 题目简述

题目在椭圆曲线上公布 $G$、$P=key\cdot G$，并用临时标量 $k$ 生成 $G'=kG$。随机消息点 $M$ 被加密为 $C=M+kP$，flag 再用 `MD5(str(M.x))` 作为 AES-ECB 密钥加密。由于曲线规模很小，可以先求椭圆曲线离散对数恢复 `key`，再消去掩码点并解密。

## 解题过程

公开参数为：

```text
p = 4559252311
a = 1750153947
b = 3464736227
G  = (2909007728, 1842489211)
P  = (1923527223, 2181389961)
G' = (1349689070, 1217312018)
C  = (662346568, 2640798701)
```

首先由 $P=key\cdot G$ 求离散对数。接着利用：

$$
kP=k(key\cdot G)=key(kG)=key\cdot G'
$$

所以消息点为：

$$
M=C-key\cdot G'
$$

完整 SageMath 脚本如下：

```python
from Crypto.Cipher import AES
from hashlib import md5

p = 4559252311
E = EllipticCurve(GF(p), [1750153947, 3464736227])

G = E(2909007728, 1842489211)
P = E(1923527223, 2181389961)
ephemeral_G = E(1349689070, 1217312018)
C = E(662346568, 2640798701)

enc = bytes.fromhex(
    "29bb47e013bd91760b9750f90630d8ef"
    "82130596d56121dc101c631dd5d88201"
    "a41eb3baa5aa958a6cd082298fc18418"
)

key = P.log(G)
assert key * G == P
print(f"key = {key}")

M = C - key * ephemeral_G
aes_key = md5(str(M[0]).encode()).digest()
flag = AES.new(aes_key, AES.MODE_ECB).decrypt(enc).rstrip(b"\x00")
print(flag)
```

运行结果为：

```text
key = 1670419487
0xGame{0b0e28c2-b36d-d745-c0be-fcf0986f316a}
```

## 方法总结

本题是小参数椭圆曲线 ElGamal 与对称加密的组合。攻击点不是 AES，而是 $P=key\cdot G$ 所在群规模过小，使离散对数可以直接求解。恢复长期私钥后，临时公钥 $G'$ 足以计算同一掩码点 $key\cdot G'=kP$。
