# week3signinRSA

## 题目简述

服务端生成标准 RSA 参数并给出 $(n,e,c)$，其中 $c=m^e\bmod n$ 是 flag 密文。菜单提供任意密文的裸 RSA 解密结果，只禁止提交整数值恰好等于 $c$ 的密文。

连接后需要先完成四字符 SHA-256 proof of work。实例地址会变化，因此脚本使用 `HOST`、`PORT` 参数，不保留历史服务 URL。

## 解题过程

未使用安全填充的 RSA 具有乘法同态。任选与 $n$ 互素的 $s$，构造

$$
c'=c\cdot s^e\bmod n.
$$

服务端解密后返回

$$
m'=(c')^d\equiv m\cdot s\pmod n.
$$

因为 $c'\neq c$，可以绕过原密文检查。再计算

$$
m=m'\cdot s^{-1}\bmod n
$$

即可恢复 flag。取 $s=2$ 最简单；使用模逆而不是直接整数除以 2，可避免对 $2m<n$ 作不必要的假设。

```python
import re
import string
from hashlib import sha256

from Crypto.Util.number import long_to_bytes
from pwn import args, remote
from pwnlib.util.iters import mbruteforce

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10002)
io = remote(HOST, PORT)


def proof_of_work():
    line = io.recvline_contains(b"sha256(XXXX+").decode().strip()
    match = re.fullmatch(
        r"sha256\(XXXX\+(.+)\) == ([0-9a-f]{64})",
        line,
    )
    if match is None:
        raise ValueError(f"无法解析 PoW：{line!r}")

    suffix, target = match.groups()
    alphabet = string.ascii_letters + string.digits
    prefix = mbruteforce(
        lambda value: sha256((value + suffix).encode()).hexdigest() == target,
        alphabet,
        4,
        method="fixed",
    )
    io.sendlineafter(b"Give me XXXX: ", prefix.encode())


proof_of_work()
io.recvuntil(b"e = ")
e = int(io.recvline())
io.recvuntil(b"n = ")
n = int(io.recvline())
io.recvuntil(b"c = ")
c = int(io.recvline())

s = 2
modified_cipher = c * pow(s, e, n) % n
if modified_cipher == c:
    raise ValueError("缩放因子没有生成不同密文")

io.sendlineafter(b"> ", b"1")
io.sendlineafter(
    b"Your cipher (in hex): ",
    format(modified_cipher, "x").encode(),
)
modified_plain = int(io.recvline().strip(), 16)

m = modified_plain * pow(s, -1, n) % n
print(long_to_bytes(m).decode())
io.close()
```

## 方法总结

漏洞根源不是 RSA 本身，而是“裸 RSA 解密 oracle + 只比较原密文”的错误访问控制。对密文乘以 $s^e$ 后，明文会同步乘以 $s$；查询变换后的密文，再乘模逆即可完整还原原明文。实际协议应使用 OAEP 等安全填充，并且不能向攻击者开放可区分的解密结果。
