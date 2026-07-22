# EzPack

## 题目简述

题目先把 42 字节的 flag 编成 336 位二进制串，再按每一位选择指数：比特为 `1` 时贡献对应的 `key[i]`，比特为 `0` 时贡献 `1`。所有贡献相加后计算

$$
c=7^s\bmod p.
$$

`output.txt` 给出 336 个背包元素和密文。解题需要先在 $7$ 生成的子群中求离散对数 $s$，再利用密钥序列的超递增结构还原比特串。

## 解题过程

记消息比特为 $b_i\in\{0,1\}$、共有 $n=336$ 位。源码中的真实指数是

$$
s=\sum_{i=0}^{n-1}
\begin{cases}
k_i,&b_i=1,\\
1,&b_i=0.
\end{cases}
$$

令 $w_i=k_i-1$，就有

$$
s=n+\sum_{i=0}^{n-1}b_iw_i.
$$

这里的常数项是 $n$，不是 $1$；遗漏每个零比特贡献的 `1` 会使后续背包目标整体出错。

密钥生成器令

$$
k_i=\sum_{j<i}k_j+r_i,
\qquad 1\le r_i\le7777.
$$

因此调整后的权重仍然超递增：

$$
w_i-\sum_{j<i}w_j=r_i+i-1>0.
$$

另一方面，$7$ 在 $\mathbb F_p^*$ 中的阶为 $(p-1)/8$。该阶完全分解为 50 个很小的素因子，最大因子仅为 `14788051`，所以 Sage 的离散对数会采用 Pohlig–Hellman 思路：在各个素数阶子群中分别求解，再用中国剩余定理合并。

本实例还有一个重要的唯一性条件：所有可能指数的最大值为 $\sum k_i$，其位数只有 348，而 $\operatorname{ord}(7)$ 有 1035 位。因此离散对数得到的剩余类就是实际整数 $s$，不存在再加子群阶的歧义。

```python
import ast

from Crypto.Util.number import long_to_bytes
from sage.all import GF, Integer, ZZ, discrete_log

p = Integer(
    2050446265000552948792079248541986570794560388346670845037360320379574792744856498763181701382659864976718683844252858211123523214530581897113968018397826268834076569364339813627884756499465068203125112750486486807221544715872861263738186430034771887175398652172387692870928081940083735448965507812844169983643977
)

with open("output.txt", "r", encoding="utf-8") as file:
    lines = file.read().splitlines()

keys = [ZZ(x) for x in ast.literal_eval(lines[0])]
cipher = Integer(lines[1])
assert len(keys) == 336

field = GF(p)
g = field(7)
h = field(cipher)
order = g.multiplicative_order()
assert order == (p - 1) // 8
assert sum(keys) < order

s = ZZ(discrete_log(h, g, ord=order))
assert g ** s == h

weights = [key - 1 for key in keys]
assert all(weights[i] > sum(weights[:i]) for i in range(1, len(weights)))

target = s - len(keys)
bits = [0] * len(keys)
for i in range(len(weights) - 1, -1, -1):
    if weights[i] <= target:
        bits[i] = 1
        target -= weights[i]

assert target == 0
message = int("".join(map(str, bits)), 2)
print(long_to_bytes(message))
```

对原附件进行复算得到：

```text
moectf{429eaa156f6961d6bc655c1887ebb779ec}
```

## 方法总结

这不是直接对原 `key` 做贪心的普通背包题。先要准确写出指数 $s=n+\sum b_i(k_i-1)$，利用平滑子群阶完成离散对数，再对调整后的超递增权重 $w_i=k_i-1$ 从大到小贪心。两个容易被忽略的检查分别是常数项 $n$ 与 $s<\operatorname{ord}(7)$；前者保证背包目标正确，后者保证离散对数没有模阶歧义。
