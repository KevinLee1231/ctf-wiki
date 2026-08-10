# matrix_equation

## 题目简述

题目生成两个 256 位素数 `k1`、`k2`，并给出：

$$
\text{temp}=p\cdot 2^{256}+q\cdot k_1+r\cdot k_2,
$$

其中 `temp` 只有 83 位。flag 由 $p+q+r$ 的 SHA-256 摘要生成：

$$
\text{flag}=\texttt{hgame\{}+\operatorname{SHA256}(\operatorname{str}(p+q+r))+\texttt{\}}.
$$

需要利用 `temp` 明显短于各系数的特征，把未知整数关系转化为格中的短向量。

## 解题过程

### 构造格

取基矩阵：

$$
M=
\begin{pmatrix}
2^{256} & 0 & 0\\
k_1 & 1 & 0\\
k_2 & 0 & 1
\end{pmatrix}.
$$

未知行向量 $(p,q,r)$ 与它相乘可得：

$$
(p,q,r)M=(\text{temp},q,r).
$$

由于 `temp` 仅有 83 位，而格基中的主尺度接近 256 位，$(\text{temp},q,r)$ 有机会成为格中的异常短向量。对 $M$ 做 LLL 约化后，第一行正好给出目标向量或它的相反数。

题目给出的常量为：

```python
k1 = 73715329877215340145951238343247156282165705396074786483256699817651255709671
k2 = 61361970662269869738270328523897765408443907198313632410068454223717824276837
hint = 83
```

SageMath 求解脚本如下：

```python
from hashlib import sha256

k1 = 73715329877215340145951238343247156282165705396074786483256699817651255709671
k2 = 61361970662269869738270328523897765408443907198313632410068454223717824276837
hint = 83

M = matrix(ZZ, [
    [2^256, 0, 0],
    [k1,    1, 0],
    [k2,    0, 1],
])

short = M.LLL()[0]

# LLL 可能返回目标向量的相反数。temp 应为正数且恰有 hint 位。
if short[0] < 0:
    short = -short

temp, q, r = map(ZZ, short)
assert temp > 0 and temp.nbits() == hint

# 求满足 (p, q, r) * M = short 的整数系数。
p, q_check, r_check = M.solve_left(short)
p, q, r = map(ZZ, (p, q_check, r_check))
assert (vector(ZZ, [p, q, r]) * M) == short

value = p + q + r
flag = "hgame{" + sha256(str(value).encode()).hexdigest() + "}"
print(temp)
print(p, q, r)
print(flag)
```

恢复出的整数为：

```text
temp = 5851117074945081723062478
p = 14012495157495443959831201
q = -9396324357950573888994599
r = -15154059265021257630097517
```

因此：

```text
hgame{3633c16b1e439d8db5accc9f602f2e821a66e6d80a412e45eb3e1048dffbb0e2}
```

官方 PDF 没有打印最终 flag；[参赛者题解](https://www.cnblogs.com/mumuhhh/p/18032304)给出了相同结果，用于补充核验。外链中的关键格构造、符号判断与最终结果均已写入正文。

## 方法总结

- 形如“大系数线性组合得到异常小结果”的整数关系，通常可以构造成格上的短向量问题。
- 构造矩阵时应让未知系数乘格基后直接出现目标小量；本题得到的正是 $(\text{temp},q,r)$。
- LLL 输出的短向量允许整体乘以 $-1$。应使用 `temp` 为正且位数等于 83 的条件消除符号歧义。
- 得到候选值后，应回代矩阵关系并重新计算 SHA-256，而不是只凭短向量的位置判断答案。
