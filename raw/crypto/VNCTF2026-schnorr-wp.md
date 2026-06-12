# Schnorr

## 题目简述

题目实现了一个 Schnorr 零知识证明服务。服务端声称只证明自己知道 flag 对应的离散对数见证 `a`，其中 `A = g^a mod p`。漏洞在于所谓的安全随机数生成器使用固定 `init_seed` 初始化；每次进程启动都会生成同样的参数和同样的第一轮承诺 `B = g^b mod p`。只要对同一个承诺给出两个不同挑战，就能利用 Schnorr 协议的 special soundness 提取见证，也就是 flag。

核心代码如下：

```python
self.secret_a = bytes_to_long(flag.encode()) % (self.p - 1)
self.A = pow(self.g, self.secret_a, self.p)

b = self._get_secure_random_bits(self.p.bit_length() - 1) % (self.p - 2) + 1
B = pow(self.g, b, self.p)
x = int(input("x = ").strip())
z = (x * self.secret_a + b) % (self.p - 1)
```

如果两次交互的 `B` 相同，并且挑战分别为 `x1`、`x2`，则有：

$$
z_1 \equiv x_1 a + b \pmod {p-1}
$$

$$
z_2 \equiv x_2 a + b \pmod {p-1}
$$

令 `x1 = 1`、`x2 = 2`，相减即可得到：

$$
a \equiv z_2-z_1 \pmod {p-1}
$$

## 解题过程

分别连接两次服务，第一次提交挑战 `1`，第二次提交挑战 `2`。因为服务端每次都用相同 seed 初始化 CSPRNG，第一轮承诺 `B` 会重复。需要记录 `p`、`B`、`z`：

```python
from pwn import *
from Crypto.Util.number import long_to_bytes

def query(x, host, port):
    r = remote(host, port)
    r.recvuntil(b"p = ")
    p = int(r.recvline())
    r.recvuntil(b"B = ")
    B = int(r.recvline())
    r.recvuntil(b"x = ")
    r.sendline(str(x).encode())
    r.recvuntil(b"z = ")
    z = int(r.recvline())
    r.close()
    return p, B, z
```

拿到两组结果后，先确认它们确实来自同一个承诺：

```python
p1, B1, z1 = query(1, HOST, PORT)
p2, B2, z2 = query(2, HOST, PORT)
assert p1 == p2
assert B1 == B2
```

然后直接提取见证：

```python
a = (z2 - z1) % (p1 - 1)
flag = long_to_bytes(a)
print(flag.decode())
```

这一步成立的原因不是离散对数被攻破，而是交互协议复用了同一个随机承诺。Schnorr 协议的安全性依赖每轮 `b` 独立随机；同一 `B` 下给出两个不同 challenge 的 response，本来就是知识提取器的标准入口。

## 方法总结

- 核心技巧：Schnorr special soundness，利用同一承诺下的两个不同挑战提取 witness。
- 识别信号：服务端“每次连接都一样”、承诺 `B` 重复、随机数来自固定 seed 或可复现 CSPRNG。
- 复用要点：对交互式零知识证明题，优先检查 nonce/commitment 是否重用；一旦同一 commitment 对应多个 challenge，就把响应方程相减消去随机数。
