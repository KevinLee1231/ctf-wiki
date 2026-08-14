# xor-with-fixed-length-key

## 题目简述

题目生成 1000 至 9999 字节长的密钥，并周期性异或 500000 字节明文。明文字节由 `os.urandom` 取模 230 得到，因此每个明文字节都落在 $[0,229]$。密钥不是直接来自安全随机源，而是由 Python `random.getrandbits` 的 MT19937 状态生成；Flag 为密钥的 SHA-256。

攻击分为两层：先利用明文字节范围恢复密钥长度和每个密钥字节的候选集合，再利用 MT19937 在 $\mathrm{GF}(2)$ 上的线性结构把候选约束成唯一内部状态。

## 解题过程

对假设的周期 $L$，同一余数位置上的所有密文字节都与同一密钥字节异或。对于密文字节 $c$，合法密钥集合为：

$$
K(c)=\{c\oplus x\mid 0\le x<230\}.
$$

真实周期应使每个位置的交集非空：

```python
from functools import reduce

possible = [{c ^ x for x in range(230)} for c in range(256)]
for key_len in range(1000, 10000):
    for pos in range(key_len):
        candidates = reduce(set.intersection,
                            (possible[c] for c in ct[pos::key_len]))
        if not candidates:
            break
    else:
        break

key_candidates = [
    reduce(set.intersection, (possible[c] for c in ct[pos::key_len]))
    for pos in range(key_len)
]
```

MT19937 的 624 个 32 位状态字共提供 19968 个状态位；固定输出长度时，`getrandbits` 可视为从状态位到密钥位的线性映射。官方脚本依次把每个状态位设为基向量，运行一次 `getrandbits(key_len*8)`，从而得到完整的布尔变换矩阵。然后在 Z3 中建立两组约束：

1. 每个符号密钥字节必须属于前面求得的候选集合；
2. 每个密钥位等于矩阵对应状态位的 XOR。

求得模型后重建 MT19937 状态，再运行相同的 `getrandbits` 生成密钥：

```python
random.setstate(recovered_state)
key = random.getrandbits(key_len * 8).to_bytes(key_len, "big")
flag = "greyhats{%s}" % hashlib.sha256(key).hexdigest()
```

结果为：

```text
greyhats{9db96ca272e09ca76491d8c2eebf1ea10b8940440c5833146b72c0db361e6236}
```

## 方法总结

- 核心技巧：用受限明文分布筛选周期 XOR 密钥，再把 MT19937 输出建模为 $\mathrm{GF}(2)$ 线性变换求状态。
- 识别信号：长密文使用固定周期密钥、明文字节范围受限、密钥来自 Python `random` 而非 `secrets`。
- 复用要点：先用便宜的分布约束大幅缩小候选，再引入 PRNG 结构；直接枚举上千字节密钥不可行。
