# DLP-2

## 题目简述

本题延续 DLP-1：客户端提交 $p,g$，服务端返回 $A=g^s\bmod p$ 以及用 `SHAKE-256(str(s))` 异或加密的 Flag。区别是服务端增加了 `isPrime(p)`，非素数模数会被拒绝。

素数检查只能保证 $\mathbb{F}_p$ 是域，却没有保证乘法群阶 $p-1$ 含有足够大的素因子。只要选择一个 $p-1$ 很光滑的素数，离散对数仍然容易求解。

## 解题过程

官方脚本仍使用：

```sage
p = factorial(320) + 1
g = 2
```

该数能够通过题目的素数检查，同时 $p-1=320!$ 完全由不超过 320 的素因子组成。于是可在 $\mathbb{F}_p^*$ 中直接调用 Sage 的离散对数实现：

```sage
from hashlib import shake_256
from pwn import remote

r = remote("HOST", PORT)
p = factorial(320) + 1
g = 2
r.sendline(str(p).encode())
r.sendline(str(g).encode())

r.recvuntil(b"output: ")
A = int(r.recvline())
r.recvuntil(b"c: ")
c = bytes.fromhex(r.recvline().decode())

secret = int(discrete_log(GF(p)(A), GF(p)(g)))
stream = shake_256(str(secret).encode()).digest(len(c))
print(bytes(a ^ b for a, b in zip(c, stream)))
```

解密结果为：

```text
greyhats{safe_prime_number_should_be_use_for_dlp_yuDmuQhX3yqVrUsd}
```

## 方法总结

- 核心技巧：素数模数并不等于安全群；攻击点是 $p-1$ 的因数结构。
- 识别信号：服务端只调用 `isPrime(p)`，却允许用户自行决定 $p$ 且没有检查大素数阶子群。
- 复用要点：DH 参数应使用经过验证的安全素数或标准化群，并验证公开值属于预期的大素数阶子群。
