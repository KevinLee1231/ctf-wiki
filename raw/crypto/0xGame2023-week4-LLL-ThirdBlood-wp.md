# LLL-ThirdBlood

## 题目简述

服务实现 DSA，但每次签名的随机数由 `getrandbits(128)` 生成，而子群阶 $q$ 为 160 位。攻击者可以对同一消息 `test` 获取多组签名；这些偏小 nonce 构成 Hidden Number Problem，可用 LLL 恢复私钥，再伪造被禁止签名的消息 `admin`。

## 解题过程

### 收集同消息签名

DSA 签名关系为：

$$
s_i\equiv k_i^{-1}(h+r_ix)\pmod q,
$$

其中 $h=\operatorname{SHA1}(\texttt{test})$，所有样本使用相同 $h$，且 $0\le k_i<2^{128}$。完成服务端的四字符 SHA-256 PoW 后，记录公开参数 $q,p,g,y$，并请求约 10 组 `test` 的 $(s_i,r_i)$。

```python
import hashlib
import itertools
import re
import string

from pwn import args, remote


def solve_pow(io):
    line = io.recvline().decode()
    match = re.search(
        r"XXXX\+([A-Za-z0-9]{16})\) == ([0-9a-f]{64})", line
    )
    suffix, target = match.groups()
    alphabet = string.ascii_letters + string.digits
    for chars in itertools.product(alphabet, repeat=4):
        prefix = "".join(chars)
        if hashlib.sha256((prefix + suffix).encode()).hexdigest() == target:
            io.sendlineafter(b"XXXX: ", prefix.encode())
            return
    raise ValueError("PoW solution not found")


io = remote(args.HOST, int(args.PORT))
solve_pow(io)

io.recvuntil(b"q=")
q = int(io.recvline())
io.recvuntil(b"p=")
p = int(io.recvline())
io.recvuntil(b"g=")
g = int(io.recvline())
io.recvuntil(b"y=")
y = int(io.recvline())

s_values, r_values = [], []
for _ in range(10):
    io.sendlineafter(b"> ", b"S")
    io.sendlineafter(b"> ", b"test")
    io.recvuntil(b"s = ")
    s_values.append(int(io.recvline()))
    io.recvuntil(b"r = ")
    r_values.append(int(io.recvline()))

print(f"q={q}\np={p}\ng={g}\ny={y}")
print(f"s={s_values}\nr={r_values}")
```

### 构造格并恢复私钥

由第 0 组签名可写出：

$$
x\equiv r_0^{-1}(s_0k_0-h)\pmod q.
$$

代入第 $i$ 组签名并整理：

$$
k_i\equiv A_i k_0+B_i\pmod q,
$$

其中

$$
A_i\equiv r_is_0(r_0s_i)^{-1}\pmod q,
$$

$$
B_i\equiv(r_0h-r_ih)(r_0s_i)^{-1}\pmod q.
$$

所有 $k_i$ 都小于 $2^{128}$，因此向量 $(k_0,k_1,\ldots,k_{n-1},k_0,2^{128})$ 相对格尺度较短。将上一步输出填入 Sage：

```sage
from Crypto.Util.number import bytes_to_long
from hashlib import sha1

q = ...
p = ...
g = ...
y = ...
s = [...]
r = [...]

h = bytes_to_long(sha1(b"test").digest())
n = len(r)
A, B = [], []

for i in range(n):
    denominator = inverse_mod(r[0] * s[i], q)
    A.append((r[i] * s[0] * denominator) % q)
    B.append(((r[0] * h - r[i] * h) * denominator) % q)

lattice = zero_matrix(ZZ, n + 2, n + 2)
for i in range(n):
    lattice[i, i] = q
    lattice[n, i] = A[i]
    lattice[n + 1, i] = B[i]

lattice[n, n] = 1
lattice[n + 1, n + 1] = 2^128
reduced = lattice.LLL()

private_key = None
for row in reduced.rows():
    if abs(row[-1]) != 2^128:
        continue
    if row[-1] < 0:
        row = -row
    k0 = ZZ(row[0]) % q
    candidate = ((s[0] * k0 - h) * inverse_mod(r[0], q)) % q
    if pow(g, candidate, p) == y:
        private_key = candidate
        break

assert private_key is not None
print(private_key)
```

公开密钥验证 `g^x mod p == y` 可以排除 LLL 输出中的错误短向量。对题目实例恢复出：

```text
27462250581507679486
```

### 伪造 `admin` 签名

```python
from Crypto.Util.number import bytes_to_long, inverse
from hashlib import sha1
from secrets import randbelow

x = 27462250581507679486
h_admin = bytes_to_long(sha1(b"admin").digest())

while True:
    nonce = randbelow(q - 1) + 1
    forged_r = pow(g, nonce, p) % q
    if forged_r == 0:
        continue
    forged_s = inverse(nonce, q) * (h_admin + forged_r * x) % q
    if forged_s != 0:
        break

assert (
    pow(g, h_admin * inverse(forged_s, q) % q, p)
    * pow(y, forged_r * inverse(forged_s, q) % q, p)
    % p
    % q
) == forged_r

io.sendlineafter(b"> ", b"C")
io.sendlineafter(b"> ", str(forged_s).encode())
io.sendlineafter(b"> ", str(forged_r).encode())
io.interactive()
```

服务端按 `s`、`r` 的顺序接收签名，验证通过后返回：

```text
0xGame{31260c7522632a69031d07133aedebfe}
```

## 方法总结

本题不是 nonce 重用，而是 nonce 的取值范围比 $q$ 小约 32 位。对同一消息收集多组签名后，可以消去私钥并把各 nonce 表示为 $k_i\equiv A_ik_0+B_i\pmod q$，再利用它们都小于 $2^{128}$ 的性质构造短向量。恢复候选私钥后必须用公开密钥验证，最后按服务端要求的 `(s,r)` 顺序提交 `admin` 签名。
