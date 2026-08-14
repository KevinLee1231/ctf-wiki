# RSA-2

## 题目简述

题目看似是标准 RSA，但 `p`、`q` 实际由 `randbits(1024)` 直接产生，并没有经过素数检查。因此 $N=pq$ 往往包含许多较小因子。题目还给出明文长度 34 字节，而 Flag 前缀 `greyhats{` 已知。

目标不必完整分解 2048 位的 $N$；只要得到足够多的互素小因子，就能恢复短明文的未知后缀。

## 解题过程

先使用 ECM 等整数分解方法获得若干 $N$ 的素因子 $r_i$。对每个因子都有：

$$
m\equiv c^{e^{-1}\bmod(r_i-1)}\pmod{r_i}.
$$

将这些余数通过 CRT 合并，可得到 $m\bmod R$，其中 $R=\prod r_i$：

```sage
partial_factors = [5, 269, 353, 4243, 24247, 1924543,
                   16744603995961, 98234797292003,
                   346338676705159]

residues = []
R = 1
for r in partial_factors:
    d = inverse_mod(e, r - 1)
    residues.append(power_mod(c, d, r))
    R *= r

m_mod_R = crt(residues, partial_factors)
```

设 Flag 总长为 $L=34$，已知前缀为 `greyhats{`。把前缀放在明文高位：

$$
m=P\cdot 256^{L-|P|}+x,
$$

其中 $x$ 是未知后缀。于是：

```sage
prefix = bytes_to_long(b"greyhats{")
x = (m_mod_R - prefix * 256^(L - len(b"greyhats{"))) % R
print(long_to_bytes(x))
```

当 $R$ 大于未知后缀的取值范围时，模 $R$ 的结果就是唯一后缀。拼回前缀得到：

```text
greyhats{too_short_for_rsa_3Sae6E}
```

## 方法总结

- 核心技巧：利用非素 RSA 因子产生的大量小因子，结合 CRT 和短明文已知前缀恢复消息。
- 识别信号：`p`、`q` 来自随机位串而非 `getPrime`，同时给出明文长度或固定格式。
- 复用要点：不一定要完全分解 $N$；只需让已知因子乘积覆盖未知明文空间即可。
