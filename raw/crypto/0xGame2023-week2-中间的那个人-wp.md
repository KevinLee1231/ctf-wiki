# 中间的那个人

## 题目简述

题目实现 Diffie-Hellman：`Alice=g^A mod p`、`Bob=g^B mod p`，共享秘密为 `g^(AB) mod p`，再经 SHA-256 作为 AES-CBC 密钥。问题在于私有指数 `A`、`B` 仅由 `getrandbits(32)` 生成，搜索空间只有 $2^{32}$，可用 baby-step giant-step/离散对数算法在约 $2^{16}$ 级别工作量下恢复其中一个指数。

## 解题过程

已知 `Alice` 和生成元 `g=2`，在模素数 $p$ 的乘法群中求：

$$
g^A\equiv Alice\pmod p,qquad 0\le A<2^{32}.
$$

SageMath 可直接求得本题的 `A=3992780394`：

```python
p = 250858685680234165065801734515633434653
g = 2
Alice = 235866450680721760403251513646370485539

A = discrete_log(Mod(Alice, p), Mod(g, p), bounds=(0, 2**32))
print(A)
```

随后用 Bob 的公钥计算共享秘密 $Bob^A\bmod p$，按源码执行 SHA-256 和 AES-CBC 解密：

```python
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from Crypto.Util.number import long_to_bytes

p = 250858685680234165065801734515633434653
Bob = 33067794433420687511728239091450927373
A = 3992780394
enc = (
    b"s\x04\xbc\x8bT6\x846\xd9\xd6\x83 y\xaah\xde"
    b"@\xc9\x17\xdc\x04v\x18\xef\xcf\xef\xc5\xfd|\x0e\xca\n"
    b"\xbd#\x94{\x8e[.\xe8\xe1GU\xfa?\xda\x11w"
)

shared = pow(Bob, A, p)
key = sha256(long_to_bytes(shared)).digest()
iv = b"0xGame0xGameGAME"
plain = AES.new(key, AES.MODE_CBC, iv).decrypt(enc)
print(unpad(plain, 16))
```

## 方法总结

Diffie-Hellman 的安全性不仅取决于大素数 $p$，还取决于私有指数的熵。128 位模数配上 32 位指数会把问题降为有界离散对数；恢复任一方私钥后即可计算同一共享秘密。实现中应使用足够长、均匀且来自密码学安全随机源的指数。
