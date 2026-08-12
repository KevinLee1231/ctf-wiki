# DownUnderCTF 2022 rsa interval oracle iii Writeup

## 题目简述

第三题提供 384 位 RSA 目标密文，允许设置 4 个区间并一次批量提交最多 4700 个查询。秘密仍只有 336 位。与逐次自适应二分相比，批量 oracle 更适合收集大量关于同一个秘密的有界线性同余式，再通过 Extended Hidden Number Problem（EHNP）恢复明文。

## 解题过程

依次添加上界 $2^{376},2^{375},2^{374},2^{373}$ 的区间。由于服务把新区间插到列表开头，最终返回编号 $j\in\{0,1,2,3\}$ 分别表示明文落入最紧的上界：

$$
0<\beta_i=r_i m\bmod N<2^{373+j}.
$$

随机选择 4700 个 $r_i$，批量查询：

```python
rs = [randint(1, N) for _ in range(4700)]
queries = [pow(r, e, N) * c for r in rs]
results = query_oracle(queries)
```

每个非 `-1` 结果都给出一个关系：

$$
\beta_i-r_i m\equiv0\pmod N,
\qquad 0<\beta_i<2^{U_i}.
$$

这些关系具有相同秘密 $m$、不同系数 $r_i$ 和已知但不完全相同的误差界，正好构成 EHNP。另一个重要先验是 $m<2^{336}$，也就是 384 位模数下最高 48 位已知为零。把 336 位未知块、每条关系的有界 $\beta_i$ 和模 $N$ 约束一同放入带权 CVP 格；足够多的有效样本会让目标附近的格点唯一对应真实秘密。

官方参数映射为：

```python
useful = [(r, 373 + result) for r, result in zip(rs, results)
          if result != -1]

xbar = 0
known_position = [0]
unknown_bits = [336]
alpha = [r for r, _ in useful]
beta_bounds = [[bits] for _, bits in useful]
```

若有效样本过少就重新连接；样本充足时求解 EHNP，按 solver 的符号约定取 `secret = -solution % N`，再验证 `pow(secret, e, N) == c`。提交后得到：

```text
DUCTF{rsa_1nt3rv4l_0r4cl3_1s_n0_m4tch_f0r_y0u!}
```

## 方法总结

批量区间泄漏可以转化为“随机倍数乘秘密后，模 $N$ 余数很小”的有界同余关系。看到大量同形关系、共同秘密和已知误差上界时，应考虑 HNP/EHNP 与 CVP，而不是逐条独立处理。明文位长先验显著提高可靠性；同时要记录每个返回编号对应的真实上界，特别注意服务插入区间时会反转顺序。
