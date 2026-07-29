# Inside

## 题目简述

题目要求提交一个 RLWE 关系的非交互式知识证明。服务端先在 secp256k1 上生成公共参考串

$$
\mathrm{crs}_i=\tau^iG,\qquad 0\le i<256,
$$

再生成公开多项式 $a,b\in\mathbb Z_{3329}[x]/(x^{256}+1)$。诚实见证应满足

$$
a(x)s(x)+e(x)\bmod (x^{256}+1)=b(x)+3329k(x),
$$

其中 $s_i,e_i\in\{-1,0,1\}$，$k$ 是整数商多项式。参赛者只能提交辅助承诺 `aux` 和三组 Maurer $\Sigma$ 协议证明；验证通过后服务端输出 flag。

问题不在 RLWE 本身，而在“整数商”被映射到椭圆曲线标量域后丢失了范围信息。证明系统只验证 $k_i\bmod q$，其中 $q$ 是 secp256k1 的群阶，却没有证明 $k_i$ 是由整数多项式除法得到的小整数。于是可以直接在模 $q$ 意义下解出一组伪造的 $k_i$。

## 解题过程

先看 `RLWEProof.prove` 对秘密向量的编码。它将 $s_i+1$ 和 $e_i+1$ 分解成两个比特，并分别承诺到 $\mathrm{crs}_i$；但对 $k_i$ 只计算

```python
aux[2][i] = k[i] * crs[i]
```

没有附加整数范围证明。`MaurerProof` 的所有响应又都会对群阶 $q$ 取模，所以这里的 $k_i$ 实际只是 $\mathbb Z_q$ 中的标量。

取

$$
s_i=e_i=-1.
$$

此时 $s_i+1=e_i+1=0$，两组比特承诺全部为无穷远点。令

$$
C=\sum_{i=0}^{255}\mathrm{crs}_i,\qquad
A_i=\sum_{j=0}^{255}a_j\mathrm{crs}_{(i+j)\bmod256}.
$$

由于下标只是循环平移，

$$
\sum_i A_i=\left(\sum_j a_j\right)C.
$$

验证器在第一组同态关系中构造的关键点为

$$
Y=3329\sum_i k_i\mathrm{crs}_i
  +\sum_i b_i\mathrm{crs}_i
  +C
  +\left(\sum_j a_j\right)C.
$$

整理每个公共参考串点的系数可得

$$
Y=\sum_i
\left(3329k_i+b_i+1+\sum_j a_j\right)\mathrm{crs}_i.
$$

因此，只需在曲线群阶 $q$ 下令

$$
k_i\equiv
-\left(b_i+1+\sum_j a_j\right)\cdot3329^{-1}
\pmod q,
$$

便能让每个系数都为零，从而得到 $Y=\mathcal O$。这组 $k_i$ 通常是接近 $q$ 的巨大标量，绝不是真实的整数商，但验证器无法区分二者。之后直接调用题目提供的 `RLWEProof.prove`，即可为这个伪造见证生成自洽的 Fiat–Shamir 证明。

下面的 Sage 求解器应与附件中的 `rlwe.py`、`sigma.py` 放在同一目录运行。它完成 PoW、获取 CRS 和公开实例、构造伪造见证，并按服务端期待的字面量格式序列化曲线点：

```python
#!/usr/bin/env sage -python
import ast
import hashlib
import itertools
import re
import string

from pwn import remote
from sage.all import ZZ

from rlwe import RLWEProof, n, q, qq
from sigma import point2tuple, tuple2point


HOST = "challenge.host"
PORT = 10000


def solve_pow(io):
    line = io.recvline_contains(b"sha256")
    match = re.search(
        rb"sha256\(XXXX \+ ([A-Za-z0-9]+)\) == ([0-9a-f]{64})",
        line,
    )
    if match is None:
        raise ValueError(f"unexpected PoW line: {line!r}")

    suffix = match.group(1)
    target = match.group(2).decode()
    alphabet = string.ascii_letters + string.digits

    for chars in itertools.product(alphabet, repeat=4):
        prefix = "".join(chars).encode()
        if hashlib.sha256(prefix + suffix).hexdigest() == target:
            io.sendlineafter(b"XXXX>", prefix)
            return
    raise RuntimeError("PoW has no solution")


def serialize_aux(aux):
    return [
        [
            [point2tuple(aux[0][i][j]) for j in range(2)]
            for i in range(n)
        ],
        [
            [point2tuple(aux[1][i][j]) for j in range(2)]
            for i in range(n)
        ],
        [point2tuple(aux[2][i]) for i in range(n)],
    ]


def serialize_proof(proof):
    return [
        (
            [point2tuple(point) for point in R],
            [int(value) for value in z],
        )
        for R, z in proof
    ]


io = remote(HOST, PORT)
solve_pow(io)

io.sendlineafter(b">", b"c")
io.recvuntil(b"crs = ")
crs_raw = ast.literal_eval(io.recvline().decode())
crs = [tuple2point(value) for value in crs_raw]

io.sendlineafter(b">", b"r")
io.recvuntil(b"st = ")
a, b = ast.literal_eval(io.recvline().decode())

sum_a = sum(ZZ(value) for value in a)
inv_qq = pow(ZZ(qq), -1, q)

s = [-1] * n
e = [-1] * n
k = [
    int((-(ZZ(b[i]) + 1 + sum_a) * inv_qq) % q)
    for i in range(n)
]

aux, proof = RLWEProof((a, b), (s, e, k)).prove(crs)
assert RLWEProof((a, b)).verify(crs, aux, proof)

io.sendlineafter(b">", b"p")
io.sendlineafter(b"aux>", repr(serialize_aux(aux)).encode())
io.sendlineafter(b"proof>", repr(serialize_proof(proof)).encode())
io.interactive()
```

在本地使用原始附件随机生成 CRS 和 RLWE 实例后，上述伪造见证已经实际通过 `RLWEProof.verify`，返回值为 `True`。远程运行时只需替换 `HOST` 与 `PORT`。

## 方法总结

本题的核心是证明语句与原始整数关系不等价。椭圆曲线上的离散对数证明只能约束标量模群阶 $q$ 的值；如果协议还需要“这是一个小整数”“这是整数除法所得的商”等语义，就必须额外提供范围证明或合适的整数承诺。

分析同态证明时，可以先主动选择让比特承诺最简单的见证，再把验证器构造的群元素按公共基点展开。本题令 $s=e=-1$ 后，复杂的卷积关系退化为 256 个线性系数，最后利用 $3329$ 在 $\mathbb Z_q$ 中可逆，逐项消去即可。修复时应对 $k$ 给出足以覆盖真实商界限的范围证明，并确保被证明的模关系与目标整数关系之间不存在环绕歧义。
