# Overflow

## 题目简述

服务端使用 ElGamal 签名，允许签任意消息，但显式拒绝字节串 `b'0xGame'`；随后只要提交一组能通过该目标消息验证的签名就返回 flag。漏洞在于签名和验证都把消息转成整数指数，而模 $p$ 的幂运算只与指数模 $p-1$ 的剩余类有关，字符串黑名单没有同步限制这个等价类。

## 解题过程

ElGamal 验证条件为

$$
g^m\equiv y^r r^s\pmod p.
$$

由于 $p$ 是素数且 $g\not\equiv0\pmod p$，费马小定理给出

$$
g^{m+p-1}\equiv g^m\pmod p.
$$

设目标整数为 $m=\operatorname{bytes\_to\_long}(\texttt{b'0xGame'})$。服务端公布公钥 `(p, g, y)` 后，构造

$$
m'=m+(p-1),
$$

并把 `long_to_bytes(m')` 交给签名接口。该字节串不等于被禁的 `b'0xGame'`，却与目标消息具有完全相同的验证左值，因此返回的 `(r, s)` 可以原样提交给目标验证。

下面的脚本同时处理源码中的 4 字节 SHA-256 proof-of-work；运行方式为 `python exp.py HOST PORT`：

```python
import ast
import hashlib
import itertools
import re
import string
import sys

from Crypto.Util.number import bytes_to_long, long_to_bytes
from pwn import remote

io = remote(sys.argv[1], int(sys.argv[2]))

# sha256(XXXX + suffix) == digest
banner = io.recvuntil(b"[+] Plz tell me XXXX: ")
match = re.search(
    rb"sha256\(XXXX\+([A-Za-z0-9]{16})\) == ([0-9a-f]{64})",
    banner,
)
suffix, expected = match.groups()

alphabet = (string.ascii_letters + string.digits).encode()
answer = None
for chars in itertools.product(alphabet, repeat=4):
    prefix = bytes(chars)
    if hashlib.sha256(prefix + suffix).hexdigest().encode() == expected:
        answer = prefix
        break

if answer is None:
    raise RuntimeError("proof-of-work not found")
io.sendline(answer)

io.recvuntil(b"Here are your public key:\n")
p, g, y = ast.literal_eval(io.recvline().decode())

target = bytes_to_long(b"0xGame")
equivalent_message = long_to_bytes(target + p - 1)
assert equivalent_message != b"0xGame"

io.recvuntil(b"> ")
io.sendline(equivalent_message)

io.recvuntil(b"r=")
r = int(io.recvline())
io.recvuntil(b"s=")
s = int(io.recvline())

io.recvuntil(b"> ")
io.sendline(str(r).encode())
io.recvuntil(b"> ")
io.sendline(str(s).encode())

print(io.recvall().decode())
```

## 方法总结

本题不是整数类型溢出，而是指数在群阶模数下的等价类绕过。凡是先把消息映射成整数、再直接放进代数运算的签名实现，都不能只对原始字符串做黑名单；应先使用抗碰撞哈希把消息映射到规定域，并严格按成熟签名标准实现和验证协议。
