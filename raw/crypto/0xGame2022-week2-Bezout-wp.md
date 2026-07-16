# week2=Bézout=

## 题目简述

服务端先给出一个 SHA-256 前缀工作量证明：已知 12 字节随机串的后 8 字节和完整哈希，需要补出前 4 字节。通过验证后，服务端随机生成 16 位整数 $a,b$，令 $c=\gcd(a,b)$，要求提交一组整数 $(s,t)$，满足

$$
as+bt=c.
$$

核心是扩展欧几里得算法，而不是枚举 $s,t$。

## 解题过程

PoW 给出的等式形如 `sha256(XXXX + suffix) == target`。`XXXX` 只有 4 个字节，字符集为大小写字母和数字，可以直接穷举。随后对 $a,b$ 执行扩展欧几里得算法。该算法在求出 $g=\gcd(a,b)$ 的同时会返回系数 $s,t$，天然满足 $as+bt=g$；源码中的 $c$ 正是这个最大公因数，因此直接提交即可。

完整交互脚本如下：

```python
import hashlib
import string

from pwn import context, remote
from pwnlib.util.iters import mbruteforce

context.log_level = "info"
HOST = "challenge-host"
PORT = 10002


def exgcd(a, b):
    if b == 0:
        return a, 1, 0
    g, x1, y1 = exgcd(b, a % b)
    return g, y1, x1 - (a // b) * y1


io = remote(HOST, PORT)

io.recvuntil(b"sha256(XXXX+")
suffix = io.recvn(8)
io.recvuntil(b") == ")
target = io.recvn(64).decode()


def check(prefix):
    digest = hashlib.sha256(prefix.encode() + suffix).hexdigest()
    return digest == target


alphabet = string.ascii_letters + string.digits
prefix = mbruteforce(check, alphabet, 4, method="fixed")
io.sendlineafter(b"XXXX :", prefix.encode())

a = int(io.recvline_contains(b"a = ").split(b"=")[1])
b = int(io.recvline_contains(b"b = ").split(b"=")[1])
c = int(io.recvline_contains(b"c = ").split(b"=")[1])

g, s, t = exgcd(a, b)
assert g == c and a * s + b * t == c
io.sendlineafter(b"s = ", str(s).encode())
io.sendlineafter(b"t = ", str(t).encode())
io.interactive()
```

服务端校验通过后返回：

```text
0xGame{you_know_how_to_interact_with_pwntools}
```

## 方法总结

本题由“短前缀 PoW + Bézout 恒等式”两部分组成。先从服务端输出中精确截取后缀与目标哈希，再穷举 4 字节前缀；数学部分用扩展欧几里得算法一次得到最大公因数及其线性组合系数。易错点是只计算 $\gcd(a,b)$ 而没有保留回代系数，或把源码提示中的变量 `y` 当成另一个输入；实际第二个输入名虽为 `t`，校验式仍是 $as+bt=c$。
