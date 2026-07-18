# superguess++

## 题目简述

服务生成 182-bit 素数 `q` 和秘密 `x < q`，给出 93 组：

```python
t_i = randbelow(q)
u_i = (x * t_i - randbelow(q >> 2)) % q
```

因此每组样本都满足一个 Hidden Number Problem 约束：

```text
e_i = (x*t_i - u_i) mod q,    0 <= e_i < q/4
```

每个样本只把乘积限制在模空间的四分之一区间，相当于约 2 bit MSB 泄露。93 个样本共提供约 186 bit 信息，刚刚超过 182-bit 未知数；普通 LLL/BKZ 很难直接捞出目标，需要在格筛法产生的中间短向量中实时识别 HNP 解。

## 解题过程

### 把非对称误差区间居中

令：

```text
C   = floor(q / 8)
z_i = e_i - C
```

则 `z_i` 近似均匀落在 `[-C,C)`。由 `u_i = t_i*x-e_i (mod q)` 可得：

```text
u_i + C = t_i*x - z_i  (mod q)
```

选一个非零 `t_0` 作为锚点，记：

```text
t'_j = t_j * t_0^(-1) mod q
a'_j = (u_j + C) - t'_j*(u_0 + C) mod q
```

消去 `x` 后：

```text
a'_j = t'_j*z_0 - z_j  (mod q)
```

问题转化为寻找一组绝对值不超过 `C` 的短整数 `z_i`。

### 构造 eliminate-alpha 格

取全部 `m=93` 个样本，令嵌入系数：

```text
E = floor(C / sqrt(3))
```

构造 `(m+1) × (m+1)` 行格：

```text
[ q*I_(m-1)                         0   0 ]
[ t'_1   t'_2   ...   t'_(m-1)      1   0 ]
[ a'_1   a'_2   ...   a'_(m-1)      0   E ]
```

最后一行减去 `z_0` 倍倒数第二行，并在前 `m-1` 个坐标加适当的 `q` 倍数后，格中存在目标向量：

```text
(-z_1, -z_2, ..., -z_(m-1), -z_0, E)
```

若把居中误差视为均匀分布，其期望平方范数约为：

```text
(m + 1) * C^2 / 3
```

这个长度与格的 Gaussian heuristic 很接近，解释了为什么只对 BKZ 约化后的基向量做检查不够：目标未必成为最短基向量，却可能在筛法数据库中作为某个中间向量出现。

### BKZ 预处理后检查 G6K 中间向量

PDF 中的 EXP 改自论文作者公开实现，流程为：

1. 用 `fpylll` 做 `BKZ-20` 预处理；
2. 用 G6K 的 `bdgl` CPU sieve 扩展数据库；有 GPU 时也可切换 GPU sieve；
3. 不把 sieve 当成只返回一个最短向量的黑盒，而是分批遍历数据库中的候选；
4. 先只计算候选的最后两个格坐标，要求嵌入坐标为 `±E` 且 `z_0` 落在允许区间；
5. 从候选 `z_0` 恢复 `x`，再用全部 93 个样本验证。

候选秘密的恢复式为：

```python
x_candidate = ((u0 + C + z0) * inverse(t0, q)) % q
```

最终判定不能依赖额外保留的验证样本，因为题目只给了刚好够建格的 93 组。应对所有原始样本检查：

```python
all((x_candidate * t - u) % q < q // 4
    for t, u in zip(T, U))
```

论文代码原本把样本拆成 `lattice/reduce/narrow/check` 四组。适配本题时应传入：

```python
n = q.bit_length()             # 182，不要保留示例中的 160
m = n // 2 + 2                 # 93
hnp = HNP(q, 2, T, U)
solver = HNPSolver(
    hnp,
    (m, 0, 0, 0),
    x=0,
    quickCheckTech=True,
    threads=<cpu-threads>,
    sieve_algorithm='bdgl',
    gpus=0,
)
solver.solve('eliminateAlpha')
```

这里的 `x=0` 是论文实现中调节格构造的参数，不是待恢复的秘密。题目有 300 秒时限，应按机器核心数设置线程并提前装好 `fpylll`、FPLLL 与 G6K。该参数下单次筛法并非必然命中；PDF 记录的成功率约为一半，失败时需重新运行，而不是误判为公式错误。

恢复到 `solver.alpha` 后直接提交，服务端比较相等即输出 flag。

### 外部实现保留说明

这篇 WP 已给出本题实际使用的 HNP 方程、格基、目标向量、候选恢复式和校验条件。若要复现完整的数据库筛选、并行筛法及通用有错样本支持，可继续使用以下稳定原始资料：

- [IACR ePrint 2024/296](https://eprint.iacr.org/2024/296)：论文提出通过参数化格构造降低维数，并结合 interval reduction、线性 predicate、pre-screening 和格筛法，缩小格攻击与傅里叶攻击在极少泄露场景下的差距；本题使用的是其中无错误、`x=0` 的简化路线。
- [JinghuiWW/ecdsa-leakage-attack](https://github.com/JinghuiWW/ecdsa-leakage-attack)：论文作者的实现，包含 `HNPSolver`、FPyLLL/BKZ、G6K CPU/GPU sieve 入口。本题需要改为 93 个样本全部建格，并把候选校验改成上面的原始 HNP 不等式。

## 方法总结

本题难点不是识别 HNP，而是参数几乎卡在信息论下界，经典“构格后取第一行”不再可靠。有效方法是先用首个样本消去秘密、把误差居中形成明确的短目标，再对 BKZ 后的格做 sieve，并在筛法过程中用廉价 predicate 主动识别目标。复用论文代码时必须重新核对参数语义、样本分组和最终校验式；直接照搬 ECDSA 示例会浪费本题本就不足的样本。
