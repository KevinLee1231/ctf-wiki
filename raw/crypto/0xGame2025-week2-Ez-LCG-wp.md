# Ez_LCG

## 题目简述

题目有两层状态机。外层是模 $2^{20}$ 的十次多项式 RNG，系数公开且 seed 由用户指定；它经过随机次数迭代后生成内层 LCG 参数 $a,b$。内层 LCG 在模 $2^{32}+1$ 下从每个 flag 字节出发迭代 1 到 1024 次并输出末状态。

虽然源码拒绝连续两次输出相等的固定点，但没有拒绝长度为 2 或 3 的短环。选择短环 seed 后，无论随机推进多少次，$a,b$ 都只能来自很小的环元素集合。

## 解题过程

先根据服务端给出的系数定义：

$$
f(x)=\sum_{i=0}^{9}c_ix^i\bmod2^{20}
$$

枚举 seed，并只跟踪少量状态，找到满足“会回到已见状态且前两次不同”的短环。提交该 seed 后，枚举环中所有 $(a,b)$。

内层递推 $x_{n+1}=ax_n+b\pmod M$ 可逆为：

$$
x_n=(x_{n+1}-b)a^{-1}\bmod M
$$

对每个密文状态最多逆推 1024 次；一旦回到字节范围，就把它作为候选明文。核心脚本如下：

```python
from ast import literal_eval
from itertools import product
from pwn import *

io = process(['python', 'task.py'])
io.recvuntil(b'Generated coefficients: ')
coefficients = literal_eval(io.recvline().decode())

outer_mod = 2**20
f = lambda x: sum(c * x**i for i, c in enumerate(coefficients)) % outer_mod

def short_cycle(seed, limit=3):
    states = [seed]
    while len(states) <= limit:
        nxt = f(states[-1])
        if nxt in states:
            return states
        states.append(nxt)

cycle = next(c for seed in range(outer_mod) if (c := short_cycle(seed)))
io.sendlineafter(b'Set seed for RNG: ', str(cycle[0]).encode())
io.recvuntil(b'Encrypted flag: ')
encs = literal_eval(io.recvline().decode())

def decrypt(a, b):
    mod = 2**32 + 1
    try:
        inv_a = pow(a, -1, mod)
    except ValueError:
        return None
    out = bytearray()
    for enc in encs:
        state = enc
        for _ in range(1024):
            if state < 256:
                out.append(state)
                break
            state = (state - b) * inv_a % mod
        else:
            return None
    return bytes(out)

for a, b in product(cycle, repeat=2):
    candidate = decrypt(a, b)
    if candidate and all(32 <= x < 127 for x in candidate):
        print(b'0xGame{' + candidate + b'}')
```

## 方法总结

- 核心技巧：用可控 seed 把外层 RNG 限制在短环，再枚举少量 LCG 参数并逆推状态。
- 识别信号：用户可控 seed、只排除固定点、参数由随机次数迭代产生但状态空间存在短周期。
- 复用要点：先攻击参数生成器而不是暴力全部 $a,b$；逆 LCG 前必须确认 $a$ 在模数下可逆，并以字节范围和 flag 格式剪枝。
