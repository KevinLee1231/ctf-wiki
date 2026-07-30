# SU_hash

## 题目简述

题目要求提交一段十六进制消息，使自定义摘要函数输出固定的 16 字节字符串 `justjusthashhash`。内部状态按下式递推：

$$
n_i=g(2n_{i-1}+m_i)\pmod{2^{383}},
$$

摘要只保留

$$
(n_i-C)\bmod 2^{128}.
$$

外层函数并非只返回最终摘要，而是把每个前缀的摘要逐一异或：

```python
def fn(msg):
    h = myhash(n0)
    ret = bytes(16)
    for b in msg:
        h.update(bytes([b]))
        ret = xor(ret, h.digest())
    return ret
```

决定性弱点是状态转移对消息字节线性，且只考察低 128 位。可以先制造“块末状态相同、块内前缀异或值不同”的碰撞块，再把目标摘要转化为 $\mathrm{GF}(2)$ 上的线性组合问题。

## 解题过程

### 用格约化构造短消息碰撞

长度为 $c$ 的消息处理完后有

$$
n_c=(2g)^c n_0+
g\sum_{i=1}^{c}m_i(2g)^{c-i}
\pmod{2^{383}}.
$$

若两个等长消息块 $m$、$m'$ 满足

$$
\sum_{i=1}^{c}(m_i-m'_i)(2g)^{c-i}
\equiv0\pmod{2^{128}},
$$

它们在题目可见的低 128 位上具有相同的块末状态。取 $c=64$，构造以 $2^{128}$ 为模数的格：

```python
c = 64
M = Matrix(ZZ, c + 1, c + 1)
for i in range(c):
    M[i, i] = 1
    M[i, c] = (2 * g) ** (c - 1 - i) % (2 ** 128)
M[c, c] = 2 ** 128

reduced = M.LLL()
```

短向量的前 64 个分量可视为对基准字节 $k$ 的小扰动。枚举合适的 $k$，只保留所有字节仍在 $[0,255]$ 内且经实际哈希验证成立的碰撞对：

```python
for row in reduced[1:-1]:
    candidate = bytes(k + int(delta) for delta in row[:-1])
    baseline = bytes([k]) * 64
    if has(candidate) == has(baseline):
        pairs.append((baseline, candidate))
```

实际验证不可省略，因为格模型只表达了截断状态的同余关系和字节“接近”约束。

### 把目标摘要化为线性方程

每一对块在结束时落到相同状态，因此后续计算可以继续共用同一状态；但块内各前缀摘要不同，会给 `fn` 的累计异或值带来一个固定差分：

$$
\Delta_i=fn(prefix\mathbin\Vert m_{i,0})
\oplus fn(prefix\mathbin\Vert m_{i,1}).
$$

依次准备至少 128 对线性独立的碰撞块。默认全选第一个块得到 `cur`，选择第二个块与否记为 $x_i\in\{0,1\}$，目标条件就是

$$
cur\oplus\bigoplus_i x_i\Delta_i
=\texttt{justjusthashhash}.
$$

把每个 16 字节值展开成 128 位向量，在 $\mathrm{GF}(2)$ 上求解：

```python
cur = bit_vector(fn(b"".join(left for left, _ in pairs), n0))
deltas = [bit_vector(delta) for delta in block_deltas]
mat = matrix(GF(2), deltas)
assert mat.rank() == 128

target = bit_vector(b"justjusthashhash")
choice = mat.solve_left(target - cur)

msg = b"".join(
    right if bit else left
    for bit, (left, right) in zip(choice, pairs)
)
assert fn(msg, n0) == b"justjusthashhash"
```

将 `msg.hex()` 提交即可通过校验。仓库中的题目环境记录的 flag 为：

```text
SUCTF{5imple_st4te_Tran3fer_w1th_s1m1lar_to_md5!!!!!}
```

## 方法总结

- 核心技巧：先用 LLL 构造截断线性状态的碰撞块，再利用块内前缀摘要的异或差分建立 128 位线性基。
- 识别信号：线性递推、$2^k$ 模数、低位截断以及“所有前缀摘要再组合”同时出现时，应分别分析最终状态与组合输出，而不是只寻找一次原像。
- 复用要点：碰撞块的价值在于保持后续状态同步，使每个块的输出差分可以独立叠加；最终必须检查差分矩阵满秩，并用题目原函数验证消息。
