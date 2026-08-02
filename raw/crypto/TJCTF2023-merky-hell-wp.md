# merky-hell

## 题目简述

题目用 48 项 Merkle–Hellman 风格背包公钥 $B_i$ 加密一个 48 位随机数：

$$
E=\sum_{i=0}^{47}b_iB_i,qquad b_i\in\{0,1\}.
$$

该随机数经字节化和 PKCS#7 补齐后作为 AES-CBC 密钥加密 flag。公开背包的密度较低，可以直接用格基规约恢复二进制选择向量。

## 解题过程

构造 49 维格：前 48 行在各自坐标放单位向量，最后一列放 $B_i$；最后一行仅在末列放 $-E$。如果某个 0/1 选择向量满足子集和，它对应的格向量最后一维为 0，且其余维很短，LLL 容易把它找出来。

```sage
M = Matrix(ZZ, 49, 49)
for i in range(48):
    M[i, i] = 1
    M[i, 48] = B[i]
M[48, 48] = -E

bits = None
for row in M.LLL().rows():
    candidate = list(row[:48])
    if row[48] == 0 and all(v in (0, 1) for v in candidate):
        bits = candidate
        break
assert bits is not None
assert sum(bit * weight for bit, weight in zip(bits, B)) == E
```

按题目从最高位到最低位的顺序把 48 个 bit 合成整数，生成 AES key 并解密：

```python
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import pad, unpad

secret = int("".join(map(str, bits)), 2)
key = pad(long_to_bytes(secret), 16)
plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(plain.decode())
```

结果为：

```text
tjctf{knaps4ck-rem0v4L0-CreEEws1278bh}
```

## 方法总结

- 低密度 0/1 子集和是典型格攻击入口；可先估算 $d=n/\log_2(\max B_i)$ 判断 LLL 是否有希望。
- 找到短向量后必须检查系数确为 0/1、末维为 0，并把子集和代回 $E$。
- 背包只负责恢复 AES 密钥材料；位序、整数到字节的转换和题目使用的补齐方式同样需要严格复刻。
