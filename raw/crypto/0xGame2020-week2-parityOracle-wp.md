# week2parityOracle

## 题目简述

这是 RSA parity oracle 的四进制变体。服务会解密任意非零密文，并只返回明文对 4 的余数：

$$
\mathcal{O}(c')=(c'^d\bmod n)\bmod4
$$

源码还特意生成满足 $n\equiv3\pmod4$ 的模数。利用 RSA 的乘法同态，把原密文依次乘以 $(4^e)^i$，就能按四等分不断缩小原明文区间。

## 解题过程

设

$$
4^i m=q_i n+r_i,\qquad0\le r_i<n
$$

oracle 返回 $r_i\bmod4$。由于 $4^i m\equiv0\pmod4$ 且 $n\equiv-1\pmod4$，有：

$$
r_i\equiv-q_i n\equiv q_i\pmod4
$$

因此返回值就是 $m/n$ 的下一位四进制区间编号：`0`、`1`、`2`、`3` 分别选择当前区间的第一、第二、第三、第四个四分位。使用 `Fraction` 保存精确有理数边界，避免原解连续整数除法造成累计舍入误差。

```python
import re
import string
from fractions import Fraction
from hashlib import sha256

from Crypto.Util.number import long_to_bytes
from pwn import *


def solve_pow(io):
    line = io.recvline_contains(b"sha256(XXXX+")
    match = re.search(
        rb"sha256\(XXXX\+([A-Za-z0-9]{16})\) == ([0-9a-f]{64})",
        line,
    )
    if match is None:
        raise ValueError(f"无法解析 proof of work: {line!r}")

    suffix, target = match.groups()

    def valid(prefix):
        digest = sha256(prefix.encode() + suffix).hexdigest().encode()
        return digest == target

    alphabet = string.ascii_letters + string.digits
    prefix = iters.mbruteforce(valid, alphabet, 4, method="fixed")
    io.sendlineafter(b"Give me XXXX: ", prefix.encode())


def oracle(io, ciphertext):
    io.sendlineafter(b"> ", b"1")
    io.sendlineafter(b"Your cipher (in hex): ", hex(ciphertext)[2:].encode())
    return int(io.recvline().strip(), 16)


def ceil_fraction(value):
    return -(-value.numerator // value.denominator)


io = remote(args.HOST, int(args.PORT))
solve_pow(io)

io.recvuntil(b"e = ")
e = int(io.recvline())
io.recvuntil(b"n = ")
n = int(io.recvline())
io.recvuntil(b"c = ")
c = int(io.recvline())
assert n % 4 == 3

lower = Fraction(0, 1)
upper = Fraction(n, 1)
current = c
multiplier = pow(4, e, n)

while upper - lower > 1:
    current = current * multiplier % n
    quarter = oracle(io, current)
    if quarter not in range(4):
        raise ValueError(f"异常 oracle 返回值: {quarter}")

    width = (upper - lower) / 4
    old_lower = lower
    lower = old_lower + quarter * width
    upper = old_lower + (quarter + 1) * width

start = max(0, ceil_fraction(lower) - 2)
stop = min(n, upper.numerator // upper.denominator + 3)

for message in range(start, stop + 1):
    if pow(message, e, n) == c:
        print(long_to_bytes(message).decode())
        break
else:
    raise ValueError("最终候选区间中没有通过重新加密验证的明文")
```

恢复结果为：

```text
0xGame{a9abdec6-7b84-4443-afb8-ee4dada8bdca}
```

## 方法总结

- 核心技巧：利用 RSA 乘法同态和模 4 解密 oracle，每轮把明文区间缩小到四分之一。
- 识别信号：可提交自选密文，服务泄漏解密结果的奇偶性或小模余数。
- 复用要点：区间映射依赖 $n\bmod4$；边界应使用精确分数，最后必须重新加密候选以消除取整误差。
