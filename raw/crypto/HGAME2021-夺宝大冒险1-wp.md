# 夺宝大冒险1

## 题目简述

服务端连续给出三组线性同余生成器（LCG）恢复任务。状态满足 $X_{n+1}\equiv AX_n+B\pmod M$：第一问已知 $A,M,X_0,X_1$ 求 $B$；第二问已知 $M$ 和四个连续状态求 $A,B$；第三问只给八个连续状态，要求恢复模数 $M$。参数由随机字节生成，可能出现不可逆或多解实例，因此求解器必须处理线性同余方程并校验候选，不能假设所有差分都与 $M$ 互素。

## 解题过程

第一问中 $X_0=123$，直接移项即可：

$$
B\equiv X_1-AX_0\pmod M.
$$

```python
def recover_b(a, modulus, x0, x1):
    return (x1 - a * x0) % modulus
```

这一步不需要求逆，因此无论 $\gcd(A,M)$ 是否为 1，$B$ 都能唯一按模 $M$ 得到。

第二问把相邻递推式相减以消去 $B$。令 $d_i=X_{i+1}-X_i$，有：

$$
d_1\equiv A d_0\pmod M,
\qquad
d_2\equiv A d_1\pmod M.
$$

若 $\gcd(d_0,M)=1$，可直接计算 $A\equiv d_1d_0^{-1}\pmod M$。一般情况下要解线性同余方程 $d_0A\equiv d_1\pmod M$：设 $g=\gcd(d_0,M)$，只有 $g\mid d_1$ 时有解；约分后先得到一个解，再枚举 $g$ 个同余类并用后续状态筛选。

```python
from math import gcd


def linear_congruence(a, b, modulus):
    """返回 a*x == b (mod modulus) 的全部模 modulus 解。"""
    common = gcd(a, modulus)
    if b % common:
        return []

    reduced_a = a // common
    reduced_b = b // common
    reduced_modulus = modulus // common
    first = (reduced_b * pow(reduced_a, -1, reduced_modulus)) % reduced_modulus
    return [first + k * reduced_modulus for k in range(common)]


def recover_a_b(modulus, states):
    x0, x1, x2, x3 = states
    candidates = []
    for a in linear_congruence(x1 - x0, x2 - x1, modulus):
        b = (x1 - a * x0) % modulus
        if all(
            (a * states[index] + b) % modulus == states[index + 1]
            for index in range(len(states) - 1)
        ):
            candidates.append((a, b))
    return candidates
```

如果仍有多个 `(A,B)` 能解释全部已知状态，而服务端要求提交隐藏的原始参数，那么现有信息在数学上不足以区分它们；最稳妥的办法是放弃该随机实例并重连，而不是随意取第一个候选。

第三问需要消去 $A$ 和 $B$。由 $d_{i+1}\equiv A d_i\pmod M$ 可得：

$$
z_i=d_{i+2}d_i-d_{i+1}^2\equiv0\pmod M.
$$

因此每个非零 $z_i$ 都是 $M$ 的倍数，所有 $|z_i|$ 的最大公约数通常就是 $M$，也可能是 $kM$。官方脚本还把不同位置的差分交叉组合，得到更多 $M$ 的倍数，以提高把额外系数约掉的概率：

```python
from functools import reduce
from itertools import combinations
from math import gcd


def recover_modulus(states):
    differences = [
        states[index + 1] - states[index]
        for index in range(len(states) - 1)
    ]

    multiples = []
    for left, right in combinations(range(len(differences) - 1), 2):
        value = (
            differences[left + 1] * differences[right]
            - differences[right + 1] * differences[left]
        )
        if value:
            multiples.append(abs(value))

    if not multiples:
        raise ValueError("该实例没有产生可用的非零消元量")
    return reduce(gcd, multiples)
```

得到的 GCD 必须进一步检查：它应大于所有输出状态，且应存在某组 $(A,B)$ 能复现整段序列。如果 GCD 明显超过题目参数的 64 位范围，说明结果仍含额外公因子；可尝试分解并测试其因子，或直接重连获取更好的随机数据。三问都通过后服务端返回动态 flag，官方 PDF 没有保存其具体值。

## 方法总结

LCG 参数恢复的核心是“差分消去 $B$，行列式消去 $A$”。第一问只是移项，不受互素性影响；第二问的除法本质是线性同余方程，非互素时会无解或多解；第三问的 GCD 只保证得到 $M$ 的倍数，不保证一次就等于 $M$。因此完整解法必须枚举、复算和识别信息不足的实例，题目使用随机字节生成参数也正是在考察失败后重试的意识。
