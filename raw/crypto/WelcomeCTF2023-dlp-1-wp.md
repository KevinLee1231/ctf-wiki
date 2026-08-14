# DLP-1

## 题目简述

服务端让客户端自行提交 Diffie–Hellman 模数 $p$ 和底数 $g$，随后生成 1024 位随机秘密 $s$，返回公开值 $A=g^s\bmod p$。Flag 使用 `SHAKE-256(str(s))` 扩展出的字节流异或加密：

```python
def encrypt_flag(secret, flag):
    key = shake_256(str(secret).encode()).digest(len(flag))
    return bytes(a ^ b for a, b in zip(key, flag))
```

漏洞在于服务端完全不检查客户端提供的群参数，因此可以主动选择一个离散对数容易求解的群。

## 解题过程

官方脚本选择：

```python
p = factorial(320) + 1
g = 2
```

这个 $p$ 满足 $p-1=320!$，其阶可以分解为大量小素因子的乘积。对这种光滑阶群，Sage 的 `discrete_log` 会使用以 Pohlig–Hellman 为核心的方法，把一个大离散对数拆成多个小素数幂阶子群中的离散对数，再通过中国剩余定理组合出 $s$。

```sage
from hashlib import shake_256
from pwn import remote

p = factorial(320) + 1
g = 2
r = remote("HOST", PORT)
r.sendline(str(p).encode())
r.sendline(str(g).encode())

r.recvuntil(b"output: ")
A = int(r.recvline())
r.recvuntil(b"c: ")
c = bytes.fromhex(r.recvline().decode())

s = int(discrete_log(GF(p)(A), GF(p)(g)))
key = shake_256(str(s).encode()).digest(len(c))
print(bytes(x ^ y for x, y in zip(c, key)))
```

恢复秘密后按服务端相同方式生成 SHAKE-256 密钥流并异或，得到：

```text
greyhats{modulus_must_be_checked_in_dlp_y632SktsY2vXTMaP}
```

## 方法总结

- 核心技巧：利用客户端可控群参数，选取 $p-1$ 光滑的模数，再用 Pohlig–Hellman 求离散对数。
- 识别信号：协议允许用户提交 $p,g$，却没有验证模数、生成元及子群阶。
- 复用要点：只检查输入格式不足以保证 DH 安全；服务端应固定可信参数，或严格校验素数、生成元和大素数阶子群。
