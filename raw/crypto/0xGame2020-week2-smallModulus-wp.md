# week2smallModulus

## 题目简述

服务把同一个 flag 整数 $m$ 对随机 64 位素数取模，每次返回一组：

$$
m\equiv a_i\pmod{n_i}
$$

源码允许重复查询，并保证 $m<2^{512}$。当不同模数两两互素、乘积 $M=\prod n_i$ 大于明文上界时，中国剩余定理可以唯一恢复 $m$。

连接服务前还要完成 proof of work：服务公开一个 20 字符串的后 16 字符和完整 SHA-256，只隐藏前 4 个字符，字符表为大小写字母和数字。

## 解题过程

原解固定收集 8 个 64 位模数，对本题实际 flag 足够，但从源码给出的唯一上界 $m<2^{512}$ 出发并不严格：每个 64 位素数都小于 $2^{64}$，所以 8 个模数的乘积仍小于 $2^{512}$。稳妥做法是继续取样，直到 $M>2^{512}$，通常需要 9 组。

```python
import re
import string
from hashlib import sha256
from math import prod

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


def get_congruence(io):
    io.sendlineafter(b"> ", b"1")
    line = io.recvline().strip()
    match = re.fullmatch(
        rb"flag mod ([0-9a-f]+) \(in hex\) : ([0-9a-f]+)",
        line,
    )
    if match is None:
        raise ValueError(f"无法解析同余数据: {line!r}")
    modulus, residue = match.groups()
    return int(modulus, 16), int(residue, 16)


def crt(residues, moduli):
    modulus_product = prod(moduli)
    value = 0
    for residue, modulus in zip(residues, moduli):
        partial = modulus_product // modulus
        value += residue * partial * pow(partial, -1, modulus)
    return value % modulus_product


io = remote(args.HOST, int(args.PORT))
solve_pow(io)

moduli = []
residues = []
while prod(moduli) <= 2**512:
    modulus, residue = get_congruence(io)
    if modulus in moduli:
        continue
    moduli.append(modulus)
    residues.append(residue)

message = crt(residues, moduli)
assert all(message % n_i == a_i for n_i, a_i in zip(moduli, residues))
print(long_to_bytes(message).decode())
```

运行时传入 `HOST=<主机> PORT=<端口>`。恢复结果为：

```text
0xGame{3a8f45be-a0cf-457e-958e-b896056841d7}
```

## 方法总结

- 核心技巧：收集同一明文在多个互素模数下的余数，再用 CRT 合并。
- 识别信号：服务反复返回 `m mod n_i`，模数随机且明文存在已知上界。
- 复用要点：必须确保模数乘积大于明文上界，并验证模数两两互素；“样本数量看起来够”不能替代唯一性条件。
