# UIUCTF 2023 Group Projection Writeup

## 题目简述

服务实现了一次经过篡改的 Diffie-Hellman 密钥交换。它在 1024 位素数模数 $p$ 下取 $g=2$、私钥 $a$，公布 $A=g^a\bmod p$；选手可以提交整数 $k$。服务随后生成私钥 $b$，并计算：

$$
B=g^b\bmod p,\quad B_k=B^k\bmod p,
$$

$$
S=B_k^a=g^{abk}\bmod p.
$$

共享秘密经过 `MD5(long_to_bytes(S))` 派生为 AES-128 密钥，flag 使用 AES-ECB 和 PKCS#7 填充加密。服务禁止几个最直接的 $k$，但没有验证 $k$ 是否会把元素投影到很小的子群。

## 解题过程

由于 $p$ 为素数，$\mathbb{Z}_p^*$ 的阶为 $p-1$。若找到 $p-1$ 的一个较小素因子 $w$，令：

$$
k=\frac{p-1}{w},
$$

则 $g^k$ 的阶至多为 $w$。服务端秘密可改写为：

$$
S=(g^{ak})^b=(A^k)^b\pmod p.
$$

因此 $S$ 只能落在由 $A^k$ 生成、大小不超过 $w$ 的子群中。无需恢复 $a$ 或 $b$，只要枚举 $i=1,\ldots,w$ 并测试 $(A^k)^i\bmod p$ 即可。若当前连接给出的 $p-1$ 在短时间内没有找到合适的小因子，就断开重连，让服务重新生成素数。

核心利用代码如下；连接和 proof-of-work 处理可按比赛环境补在 `remote` 前后：

```python
import hashlib
import itertools
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad
from pwn import remote


def find_small_factor(n, limit=100_000):
    for candidate in itertools.chain([2], range(3, limit, 2)):
        if n % candidate == 0:
            while n % candidate == 0:
                n //= candidate
            # 避开服务明确禁止的 (p - 1) / 2，并让枚举规模可控
            if candidate >= 100:
                return candidate
    return None


io = remote("group-projection.chal.uiuc.tf", 1337)
io.recvuntil(b"g = ")
g = int(io.recvline())
io.recvuntil(b"p = ")
p = int(io.recvline())
io.recvuntil(b"A = ")
A = int(io.recvline())

w = find_small_factor(p - 1)
if w is None:
    raise RuntimeError("本轮没有快速找到合适的小因子，请重连")

k = (p - 1) // w
io.sendlineafter(b"Choose k = ", str(k).encode())
io.recvuntil(b"c = ")
ciphertext = long_to_bytes(int(io.recvline()))

Ak = pow(A, k, p)
for i in range(1, w + 1):
    shared = pow(Ak, i, p)
    key = hashlib.md5(long_to_bytes(shared)).digest()
    plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
    try:
        plaintext = unpad(plaintext, 16)
    except ValueError:
        continue
    if plaintext.startswith(b"uiuctf{"):
        print(plaintext.decode())
        break
```

成功枚举后得到：

```text
uiuctf{brut3f0rc3_w0rk3d_b3f0r3_but_n0t_n0w!!11!!!}
```

## 方法总结

漏洞不在 AES 或 MD5 本身，而在协议允许攻击者控制指数 $k$，却没有检查变换后的群元素阶。选取 $k=(p-1)/w$ 会把共享秘密限制到至多 $w$ 个候选中，把原本困难的离散对数问题降为小规模枚举。处理此类协议时，应验证接收元素属于期望的大阶子群，并拒绝单位元和所有小子群元素，而不能只屏蔽几个显眼的参数值。
