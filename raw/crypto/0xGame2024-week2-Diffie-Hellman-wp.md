# Diffie-Hellman

## 题目简述

服务公开 $(q,g)$ 和 Alice 公钥 $A=g^a\bmod q$，再让客户端自行提交 Bob 公钥 $B$。程序没有验证 $B$ 是否由未知私钥生成，因此直接令 $B=g$，共享密钥就退化为 $g^a=A$，可由公开信息计算并解密 AES-ECB 密文。

## 解题过程

正常情况下：

$$
A=g^a\pmod q,\qquad B=g^b\pmod q
$$

Alice 计算的共享密钥为：

$$
K=B^a=g^{ab}\pmod q
$$

本题允许任意提交 `Bob_PubKey`。选择 $B=g=g^1$ 后：

$$
K=B^a=g^a=A\pmod q
$$

所以无需计算离散对数，服务已经把共享密钥以 Alice 公钥的形式给出。源码再使用：

```python
AES.new(md5(str(Share_Key).encode()).digest(), AES.MODE_ECB)
```

完整交互脚本如下：

```python
from hashlib import md5, sha256
from itertools import product
from string import ascii_letters, digits

from Crypto.Cipher import AES
from pwn import remote


io = remote("HOST", PORT)

# 4 字符 SHA-256 PoW
io.recvuntil(b"sha256(XXXX+")
suffix = io.recvuntil(b")", drop=True)
io.recvuntil(b"== ")
target = io.recvline().strip().decode()

for chars in product(ascii_letters + digits, repeat=4):
    prefix = "".join(chars).encode()
    if sha256(prefix + suffix).hexdigest() == target:
        io.sendlineafter(b"XXXX: ", prefix)
        break

io.recvuntil(b"Share (q,g) : (")
q, g = map(int, io.recvuntil(b")", drop=True).decode().split(","))

io.recvuntil(b"Alice_PubKey : ")
alice_public = int(io.recvline())

# 令 Bob 私钥等效为 1，公钥直接提交 g
io.sendlineafter(b"> ", str(g).encode())

io.recvuntil(b"Alice tell Bob : ")
ciphertext = bytes.fromhex(io.recvline().strip().decode())

key = md5(str(alice_public).encode()).digest()
plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(plaintext.rstrip(b"\x00").decode())
io.close()
```

解密得到：

```text
0xGame{107c7960-d339-48b5-92b9-d59ad5644cf6}
```

## 方法总结

本题并未破坏离散对数假设，而是允许攻击者提交未经验证的 DH 公钥。选择已知指数 1 的公钥 $g$，会让共享密钥直接等于公开的 Alice 公钥；协议实现应验证对端公钥的范围、群成员关系，并使用经过认证的密钥交换流程。
