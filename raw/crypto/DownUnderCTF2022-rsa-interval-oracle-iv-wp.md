# DownUnderCTF 2022 rsa interval oracle iv Writeup

## 题目简述

复仇版把四个区间固定为：

$$
(0,2^{373}),\ (0,2^{374}),\ (0,2^{375}),\ (0,2^{376}),
$$

并且只允许发送一次、最多包含 4700 个密文的 oracle 请求。目标仍是恢复 384 位 RSA 模数下的 336 位秘密。一次查询无法进行自适应二分，但可以并行采集 EHNP 所需的全部有界同余样本。

## 解题过程

随机生成 4700 个乘数 $r_i$，利用 RSA 乘法性质提交：

$$
c_i=r_i^e c\bmod N.
$$

解密结果为 $r_i m\bmod N$。返回区间编号 $j$ 时可写成：

$$
\beta_i-r_i m+k_iN=0,
\qquad 0<\beta_i<2^{373+j},
$$

其中 $k_i$ 是未知整数。返回 `-1` 的样本没有足够小的余数，直接丢弃。通常需要至少约 50 个有效样本；若某次实例的有效样本太少，只能重新连接并重新采样。

将这些关系连同 $0\le m<2^{336}$ 的先验编码为 EHNP：主未知块是 336 位的 $m$，每条关系还有一个受不同位数上界约束的 $\beta_i$。构造按各未知量范围缩放的格后，以每个有界区间的中点作为 CVP 目标，近似最近格点会泄露 $m$。

```python
rs = [randint(1, N) for _ in range(4700)]
cts = [pow(r, e, N) * c for r in rs]
res = query_once(cts)

useful = [(r, 373 + j) for r, j in zip(rs, res) if j != -1]
if len(useful) < 55:
    retry_instance()

secret = recover_ehnp(
    modulus=N,
    secret_bits=336,
    multipliers=[r for r, _ in useful],
    remainder_bits=[u for _, u in useful],
)
assert pow(secret, e, N) == c
```

恢复并提交秘密后得到：

```text
DUCTF{rsa_1nt3rv4l_0r4cl3_1s_s3ri0usly_n0_m4tch_f0r_y0u...94b2a797eb5e0105}
```

## 方法总结

本题说明一次性 oracle 仍可能泄漏足够的信息：关键不是交互次数，而是一次能否批量取得大量关于同一秘密的有界关系。固定的分层区间还为每个样本提供了不同精度的误差界，适合使用 EHNP 的加权格模型。实现时必须在发送唯一一次查询前生成全部样本，并用公钥重新加密验证格算法给出的候选。
