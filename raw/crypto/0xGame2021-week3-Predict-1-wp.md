# week3Predict 1

## 题目简述

服务端使用未知参数的线性同余生成器

$$
x_{i+1}\equiv ax_i+b\pmod p,
$$

其中 $a,p$ 是 90 bit 素数。初始资金允许先故意猜错获取若干连续状态，恢复参数后再持续预测并把资金提升到 200 以获得 flag。

## 解题过程

令 $\Delta_i=x_{i+1}-x_i$，消去常数 $b$ 可得

$$
\Delta_{i+1}\equiv a\Delta_i\pmod p.
$$

继续消去 $a$：

$$
z_i=\Delta_{i+1}^2-\Delta_i\Delta_{i+2}\equiv0\pmod p.
$$

因此多个非零 $z_i$ 的最大公因数含有 $p$。取足够多的连续输出后，通常可直接得到素数 $p$；若结果仍带小因子，应增加样本或剥离额外因子。随后计算

$$
a\equiv\Delta_{i+1}\Delta_i^{-1}\pmod p,
\qquad
b\equiv x_{i+1}-ax_i\pmod p.
$$

完整交互脚本如下，连接信息需替换为当前题目实例：

```python
from math import gcd
from Crypto.Util.number import inverse, isPrime
from pwn import remote

HOST = "challenge.example"
PORT = 0
io = remote(HOST, PORT)

states = []
for _ in range(16):
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b">", b"-1")  # 状态非负，保证猜错
    io.recvuntil(b"Right Answer is ")
    states.append(int(io.recvline()))

deltas = [states[i + 1] - states[i] for i in range(len(states) - 1)]
relations = [
    abs(deltas[i + 1] ** 2 - deltas[i] * deltas[i + 2])
    for i in range(len(deltas) - 2)
]

p = 0
for value in relations:
    if value:
        p = gcd(p, value)
if not isPrime(p):
    raise ValueError("gcd 仍含额外因子，请增加状态样本或分解该 gcd")

a = deltas[1] * inverse(deltas[0], p) % p
b = (states[1] - a * states[0]) % p
assert all((a * states[i] + b) % p == states[i + 1]
           for i in range(len(states) - 1))

x = states[-1]
for _ in range(205):
    x = (a * x + b) % p
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b">", str(x).encode())

print(io.recvrepeat(1).decode())
```

服务端每次随机生成 flag 的末 10 个十六进制字符，格式为：

```text
0xGame{86767788-6000-7608-6777-54[十位十六进制]}
```

原 WP 记录的一次实际结果是 `0xGame{86767788-6000-7608-6777-5454a581d836}`，它只是动态实例，不是固定答案。

## 方法总结

未知模数 LCG 可通过连续差分构造恒为 $p$ 倍数的行列式关系，再对多个关系求 `gcd`。恢复参数后必须用全部已知状态验证递推式，避免把带额外因子的 `gcd` 误当作模数。
