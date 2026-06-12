# Flux

## 题目简述

题目附件包含一个二次递推生成器和一个自定义 hash。`Flux.next()` 在有限域模数 `n` 下更新状态，参数 `a,b,c` 和初始状态未知；程序把 `shash("Welcome to HGAME 2026!", key)` 作为初始状态，输出 4 个连续递推值和模数 `n` 到 `data.txt`，最终 flag 是 `shash("I get the key now!", key)` 的十六进制形式。

```python
class Flux:
    def __init__(self, n, x):
        self.n = n
        self.a = random.randint(1, n-1)
        self.b = random.randint(1, n-1)
        self.c = random.randint(1, n-1)
        self.x = x

    def next(self):
        self.x = (self.a * self.x ** 2 + self.b * self.x + self.c) % self.n
        return self.x

def shash(value: str, key: int) -> int:
    mask = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    x = (ord(value[0]) << 7) & mask
    for c in value:
        x = (key * x) & mask ^ ord(c)
    x ^= len(value) & mask
    return x
```

`data.txt` 的形态是一行 4 个递推输出和一行 260 bit 左右的素数模数。

## 解题过程

题目实际分成两步。第一步，`data.txt` 给出了 4 个连续状态和模数 `n`，它们满足二次递推：

```text
k[i+1] = a * k[i]^2 + b * k[i] + c (mod n)
```

把未知初始状态和系数写成多项式变量，使用 Groebner basis 可以直接解出 `a,b,c` 等参数：

```sage
# Sage
P.<d,a,b,c> = GF(n)[]
k = [d, k1, k2, k3, k4]
eqs = [k[i+1] - (a * k[i]**2 + b * k[i] + c) for i in range(4)]
G = ideal(eqs).groebner_basis()

vals = {}
for poly in G:
    if len(poly.variables()) == 1:
        v = poly.variables()[0]
        roots = poly.univariate_polynomial().roots()
        if roots:
            vals[v] = roots[0][0]
```

第二步，题目还有一个自定义 `shash`，可以用 z3 把 key 当成定长 bit-vector 反求：

```python
from z3 import *

def shash(value: str, key) -> int:
    length = len(value)
    if length == 0:
        return 0
    mask = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    x = (ord(value[0]) << 7) & mask
    for c in value:
        x = (key * x) & mask ^ ord(c)
    x ^= length & mask
    return x

key = BitVec('key', 70)
solver = Solver()
solver.set("timeout", 60000)
solver.add(simplify(shash("Welcome to HGAME 2026!", key)) == TARGET_HASH)

if solver.check() == sat:
    recovered_key = solver.model()[key].as_long()
    magic_word = "I get the key now!"
    flag = "VIDAR{" + hex(shash(magic_word, recovered_key))[2:] + "}"
    print(flag)
```

这里真正要保留的是递推关系和 `shash` 结构；有限域常量、状态样本和目标 hash 属于题目数据，正文只描述其形态，不需要把 `data.txt` 的大整数全文贴入 WP。

## 方法总结

有限域递推题先把未知参数写成多项式方程组，样本足够时可以直接用 Groebner basis 消元。自定义 hash 若只涉及固定宽度整数运算，可交给 SMT 求解；WP 中应保留递推式和 hash 结构，而不是只贴结果。
