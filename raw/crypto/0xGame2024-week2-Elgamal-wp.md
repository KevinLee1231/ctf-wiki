# Elgamal

## 题目简述

服务给出消息 `Welcome_to_0xGame2024_Crypto` 的一组 ElGamal 签名，要求为另一条消息提供有效签名。验签函数没有检查 $r$ 是否位于标准范围 $0<r<q$，因此可以利用指数按 $q-1$ 取模、底数按 $q$ 取模的差异，通过 CRT 把原签名缩放成新消息的签名。

## 解题过程

验签条件是：

$$
y^r r^s\equiv g^m\pmod q
$$

其中 $m=\operatorname{SHA256}(\text{msg})$。设目标消息哈希为 $m'$，选择：

$$
u\equiv m' m^{-1}\pmod{q-1}
$$

将原验签等式两边同时取 $u$ 次幂：

$$
y^{ur}r^{us}\equiv g^{um}\equiv g^{m'}\pmod q
$$

若构造的新参数满足：

$$
r'\equiv ur\pmod{q-1},\qquad r'\equiv r\pmod q
$$

以及：

$$
s'\equiv us\pmod{q-1}
$$

则 $y^{r'}\equiv y^{ur}\pmod q$，同时 $r'\equiv r\pmod q$，新签名就会通过。由于 $q$ 与 $q-1$ 互质，可用 CRT 求出通常远大于 $q$ 的 $r'$；缺失的范围检查正是漏洞成立条件。

下面脚本读取当前实例给出的 $q,r,s$，为消息 `Test` 生成签名：

```python
from hashlib import sha256
from math import gcd


def hash_int(message):
    return int.from_bytes(sha256(message).digest(), "big")


def crt_pair(a1, m1, a2, m2):
    # x = a1 (mod m1), x = a2 (mod m2)
    step = (a2 - a1) * pow(m1, -1, m2) % m2
    return (a1 + step * m1) % (m1 * m2)


q = int(input("q = "))
r = int(input("r = "))
s = int(input("s = "))

original = b"Welcome_to_0xGame2024_Crypto"
forged = b"Test"
phi = q - 1

m = hash_int(original)
m_prime = hash_int(forged)
assert gcd(m, phi) == 1

u = m_prime * pow(m, -1, phi) % phi
r_prime = crt_pair(u * r % phi, phi, r % q, q)
s_prime = u * s % phi

print("message hex =", forged.hex())
print("r =", r_prime)
print("s =", s_prime)
```

依次提交 `54657374`、脚本输出的 $r'$ 和 $s'$，服务返回：

```text
flag : 93e9adb8-8a6d-4517-9c61-13081c413e41
```

源码中的 `secret.py` 本身没有添加 `0xGame{}` 外壳，上述内容按服务真实输出保留。

## 方法总结

漏洞来自验签输入域不完整：$r$ 既作为指数又作为模 $q$ 的底数，却没有被限制在标准区间。通过同时满足模 $q-1$ 和模 $q$ 的两个同余条件，可以把已有签名线性缩放到新消息；修复时至少要检查 $0<r<q$、$0<s<q-1$。
