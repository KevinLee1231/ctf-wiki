# yet another shuffled MT game

## 题目简述

本题沿用 `yet another MT game` 的 512-bit secret 与 Sage/GMP MT19937，但只允许 3 次查询，未破损状态下累计最多泄露 16400 bit。一旦累计量首次超过上限，服务会先生成矩阵，再用 Sage 的 `shuffle()` 打乱每个元素的 bit 和整个输出列表。`random_matrix` 的小模数路径使用 GMP-MT，而 `shuffle()` 使用另一套懒初始化的 Python MT；利用点正是两套 PRNG 之间的初始化依赖。

## 解题过程

### 先区分 GMP-MT 与 PY-MT

`set_random_seed(secret)` 立即初始化 Sage 的 GMP randstate，却不会立刻创建 Python `random.Random`。第一次调用 `sage.misc.prandom` 或 `shuffle()` 时，`python_random()` 才执行：

```python
rand = random.Random()
rand.seed(int(ZZ.random_element(1 << 128)))
```

这个 128-bit Python seed 来自 GMP randstate。对应 Sage/GMP 实现还会额外消耗一个 32-bit word，所以初始化 PY-MT 总共推进 GMP MT 5 个 32-bit 输出。初始化完成后，两套 MT 独立演化。

源码的查询顺序也很关键：

```python
leaked += nbits * nrow * ncol
if leaked > 16400 and not IS_BROKEN:
    IS_BROKEN = True
outs = random_matrix(...)
if IS_BROKEN:
    outs = [shuffle_bits(x) for x in outs]
    shuffle(outs)
```

也就是说，第一次越过上限的那次输出已经会被置乱。

### 比赛中采用的非预期解：先恢复 Python seed

第一问提交：

```text
mod = 2^32 - 2,  nrow = 1,  ncol = 300
```

这只计入 `32*300=9600` bit，不会触发破损。该模数走通用矩阵路径，元素来自 Python `random.randint(0, mod-1)`。这也是 PY-MT 的第一次使用，因此在生成 300 个元素前，Sage 先从 GMP-MT 取得 128-bit 整数种子。

由于上界非常接近 `2^32`，每个元素几乎总是 Python MT 的一个完整 32-bit 输出。利用 CPython 的整数 seed 初始化关系，不必收集 624 个完整输出；不到 300 个输出便足以反推短至 128 bit 的 seed。PDF 使用的辅助实现提供两个关键接口：

```python
breaker.get_required_output_indices_for_integer_seed_recovery(128, True)
breaker.recover_all_integer_seeds_from_few_outputs(
    128, selected_outputs, True
)
```

它先计算哪些输出位置足以约束 128-bit 整数 seed，再逆向 CPython 的 `init_by_array` 初始化并枚举候选。实现见 [python_random_breaker.py](https://github.com/Aeren1564/CTF_Library/blob/master/CTF_Library/Cryptography/MersenneTwister/python_random_breaker.py)；正文已经说明它在本题中的输入、输出和用途。

恢复后必须用候选 `Random(seed)` 重放第一问，确认 300 个值完全一致。更稳妥的推进方式是调用与 Sage 相同的 `randrange(2**32-2)`，而不是假设永远没有 rejection sampling；若 seed 恢复阶段恰遇极小概率的拒绝取样导致输出错位，直接重连即可。

### 重放 PY-MT 并逆置乱 GMP 输出

第二问提交：

```text
mod = 2,  nrow = 1,  ncol = 20000
```

累计泄露超过上限，于是服务先生成 20000 个 GMP-MT 最低位，再置乱。每个元素只有 1 bit，`shuffle([bit])` 不进行交换，也不消耗随机数；真正起作用的只有最后一次 `shuffle(outs)`。

用恢复出的 Python RNG 重放第一问消耗后，对索引列表执行同一 shuffle：

```python
indices = list(range(20000))
r.shuffle(indices)

real = [None] * 20000
for shuffled_pos, original_pos in enumerate(indices):
    real[original_pos] = observed[shuffled_pos]
```

此时 `real` 就是按真实时间顺序排列的 20000 个 GMP-MT LSB。第三次查询可以发送一个最小合法请求，使服务进入 secret 提交阶段；它不会影响已经收集的方程。

### 对齐额外的 160 bit 消耗并恢复 GMP seed

后半部分与未置乱版本相同：用 624 个符号 32-bit word 模拟 GMP MT，在 GF(2) 上加入固定状态位和 20000 个最低位方程。但输出起点多了 PY-MT 初始化消耗，必须在 2000 次 GMP warm-up 后再推进 160 bit：

```python
align_gmp_warmup(symbolic_rng, 2000)
symbolic_rng.getrandbits(128)
symbolic_rng.getrandbits(32)   # 合计 5 个 MT word

for bit in real:
    equations.append((symbolic_rng.getrandbits(32) & 1) ^ bit)
```

解出 19937-bit GMP 初始化值后，仍使用上一题的映射：

```text
P = 2^19937 - 20023
E = 1074888996
seed2 = (secret + 2)^E mod P
```

先求 `E/12` 在 `(P-1)/12` 下的逆，再对结果做精确整数 12 次根，减 2 并编码为固定 64 字节，即可提交 secret。

### 官方预期路线

仓库单题 WP 还给出了不依赖 PY-MT 短 seed 逆向工具的预期思路。先取得 16400 个确定的 GMP 输出方程 `M*s=y`；方程不足以唯一确定 19937-bit 状态，但可以求一个特解和核空间基：

```text
s = s0 + c1*k1 + ... + cd*kd
```

继续符号模拟 GMP MT，未来每个输出位都可写成残余变量 `c_i` 的线性式。寻找一个偏移，使即将用于 PY-MT 懒初始化的连续 160 bit 全部退化为常数；在该位置首次触发 shuffle，就能预测其 128-bit seed。之后重建所有置换，再把更多乱序输出还原为确定的 GMP 方程，最终解出状态。

预期解和比赛解利用的是同一根因：PY-MT 虽与 GMP-MT 独立运行，但 seed 来自 GMP-MT 的可预测状态。区别只是前者从欠定 GMP 方程中寻找“未来常量窗口”，后者先直接泄露 PY-MT 输出并逆向其短 seed。

## 方法总结

看到“MT 输出 + shuffle”不能只把置乱当信息损失；应先查 shuffle 使用哪套 PRNG、何时初始化、seed 从哪里来以及具体消耗多少底层输出。本题通过第一问选择通用矩阵主动暴露 PY-MT，再通过第二问选择 1-bit 元素让逐元素 shuffle 退化为空操作，最终把置乱恢复成普通的 GMP MT 线性方程。PRNG 分层与懒初始化往往比单个 MT19937 本身更值得审计。
