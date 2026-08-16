# Diffie_3ilman

## 题目简述

服务端生成素数 `p`、底数 `g = 2`、私钥 `a` 和公钥 $A=g^a\bmod p$，随后允许玩家提交一个指数 `k`。服务器另取私钥 `b`，计算 $B=g^b\bmod p$，再以 $B^{ka}\bmod p$ 作为共享秘密派生 AES 密钥。虽然 `k = 1` 和 `k = p - 1` 被禁用，但仍可把运算压缩到只有两个元素的子群。

## 解题过程

### 选择半群阶指数

提交：

$$
k=\frac{p-1}{2}
$$

根据欧拉判别准则，对任意非零 $B$ 都有：

$$
B^{(p-1)/2}\equiv \left(\frac{B}{p}\right)\in\{1,-1\}\pmod p
$$

于是服务器的共享秘密为：

$$
shared=(B^k)^a\bmod p\in\{1,p-1\}
$$

官方说明只讨论了 `shared = 1` 的情况，但它并非每次都必然成立。更稳妥的做法是对 `1` 和 `p - 1` 两个候选都派生 AES key，并用 PKCS#7 padding 与 flag 前缀判断正确结果。

```python
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad
from hashlib import md5
from pwn import remote

io = remote("host", 8000)
io.recvuntil(b"g = ")
g = int(io.recvline())
io.recvuntil(b"p = ")
p = int(io.recvline())
io.recvuntil(b"A = ")
A = int(io.recvline())

k = (p - 1) // 2
io.sendlineafter(b"Choose k = ", str(k).encode())
io.recvuntil(b"c = ")
c = int(io.recvline())
ciphertext = long_to_bytes(c, blocksize=16)

for shared in (1, p - 1):
    key = md5(long_to_bytes(shared)).digest()
    try:
        plain = unpad(AES.new(key, AES.MODE_ECB).decrypt(ciphertext), 16)
        if b"shellmates{" in plain:
            print(plain)
    except ValueError:
        pass
```

得到：

```text
shellmates{D0_n0t_L3t_Str4ngers_Pl4y_w1th_D1ffie_H3llman_p4ram$}
```

## 方法总结

- 核心技巧：通过可控指数把 Diffie-Hellman 共享秘密限制在阶为 2 的集合中，再枚举全部候选 AES key。
- 识别信号：协议允许对方选择未经子群检查的指数或群元素时，应检查小子群约束和退化共享秘密。
- 复用要点：`k=(p-1)/2` 只保证结果为 $\pm1$，不保证恒为 1；WP 和 solver 都应覆盖两个分支。
