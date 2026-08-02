# TSGCTF2024 Mystery of Scattered Key

## 题目简述

题目生成两个 1024 位 RSA 强素数 $p,q$，计算 $N=pq$，再用 $e=65537$ 加密 flag。随后把每个素数按大端序切成 64 个 16 位块，并分别随机打乱：

```python
p_bytes = p.to_bytes(128, 'big')
q_bytes = q.to_bytes(128, 'big')

p_splitted = [
    int.from_bytes(p_bytes[i:i+2], 'big')
    for i in range(0, 128, 2)
]
q_splitted = [
    int.from_bytes(q_bytes[i:i+2], 'big')
    for i in range(0, 128, 2)
]
shuffle(p_splitted)
shuffle(q_splitted)
```

附件给出 $N$、密文 $c$ 和两组无序块。目标是恢复块的正确顺序，重新得到 $p,q$，再做标准 RSA 解密。

## 解题过程

### 1. 从最低 16 位逐步恢复两个因子

令基数 $B=2^{16}$，把两个素数写成：

$$p=\sum_{i=0}^{63}p_iB^i,\qquad q=\sum_{i=0}^{63}q_iB^i$$

虽然附件中的块原本按大端切分，但打乱后只剩一个多重集合；恢复时应从最低 limb 开始。若已经确定 $p_0\ldots p_{i-1}$ 和 $q_0\ldots q_{i-1}$，尝试把剩余块 $a,b$ 放到第 $i$ 位，只需检查：

$$
(p_{<i}+aB^i)(q_{<i}+bB^i)
\equiv N\pmod{B^{i+1}}
$$

高于第 $i$ 位的未知块都是 $B^{i+1}$ 的倍数，不影响这个同余式。因此可逐层筛选候选对。

官方 solver 把两组块随机打乱后，每层取第一个局部匹配。该策略并不完备：实测一次在第 9 层走入死路；重新打乱后虽然能成功，但可靠实现应使用回溯，并在第 64 层验证 $pq=N$：

```python
B = 1 << 16

def recover(i, p, q, p_parts, q_parts):
    if i == 64:
        return (p, q) if p * q == N else None

    modulus = B ** (i + 1)
    candidates = []
    for a in p_parts:
        for b in q_parts:
            pp = p + a * (B ** i)
            qq = q + b * (B ** i)
            if pp * qq % modulus == N % modulus:
                candidates.append((a, b, pp, qq))

    for a, b, pp, qq in candidates:
        next_p = p_parts.copy()
        next_q = q_parts.copy()
        next_p.remove(a)
        next_q.remove(b)
        result = recover(i + 1, pp, qq, next_p, next_q)
        if result is not None:
            return result
    return None
```

也可以保留官方随机贪心并在失败时重启，但回溯更能说明算法为何正确。恢复结果满足附件给出的 $N$：

```text
p = 13384607956703335629...917259257926213
q = 15135551837276512049...789976020567069
```

### 2. 标准 RSA 解密

得到因子后计算：

$$\varphi(N)=(p-1)(q-1)$$

$$d=e^{-1}\bmod\varphi(N)$$

$$m=c^d\bmod N$$

```python
phi = (p - 1) * (q - 1)
d = pow(65537, -1, phi)
m = pow(c, d, N)
flag = m.to_bytes((m.bit_length() + 7) // 8, 'big')
```

对附件 `output.txt` 实际运行恢复程序，得到：

```text
TSGCTF{Yasu_is_the_culprit_4977d14abf9a4fad90d87046d2ee7e7d}
```

仓库中的 `writeup.en.md` 仍标记为 WIP，并列出另一个短 flag；它与 `src/challenge.py`、`output.txt` 和实测解密结果不一致，因此不能采用。

## 方法总结

本题泄露的不是连续高位或低位，而是两个素数各自的全部 16 位 limb 多重集合。乘积模 $B^{i+1}$ 只依赖因子的前 $i+1$ 个低位 limb，使得组合问题可以从低位逐层约束，而不必枚举 $64!$ 种排列。局部同余可能有多个候选，所以最终实现应回溯或至少验证完整乘积；恢复 $p,q$ 后，剩余部分就是标准 RSA 私钥计算。
