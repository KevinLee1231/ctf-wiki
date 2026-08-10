# LLLCG Revenge

## 题目简述

Revenge 版补上了上一题缺失的括号，状态重新在素数模数 $p\approx2^{360}$ 下更新。flag 整数仍作为未知乘数 $a$，每轮噪声满足 $|r_i|\le 2^{340}$。给出 40 个连续状态，目标是把带有约 20 位高位信息的模线性关系转成隐藏数问题，用 LLL 与 Babai 最近平面法恢复 $a$。

## 解题过程

修正后的递推为：

```python
self.seed = (
    self.seed * self.a + randint(-(2**340), 2**340)
) % self.n
```

令公开输出为 $s_0,s_1,\ldots,s_{39}$，则对 $i=0,\ldots,38$：

$$
s_{i+1}\equiv a s_i+r_i\pmod p,
\qquad |r_i|\le 2^{340}.
$$

等价地，存在整数 $k_i$ 使：

$$
a s_i+k_i p=s_{i+1}-r_i.
$$

因为 $p$ 约为 360 位，而误差只有 340 位，$s_{i+1}$ 给出了 $a s_i\bmod p$ 的约 20 个最高有效位。39 组样本足以建立 HNP 格。

构造 $40$ 维格：前 39 个基向量分别在对角线上放置 $p$，最后一个基向量为

$$
(s_0,s_1,\ldots,s_{38},1/p).
$$

它的某个整数线性组合接近目标向量

$$
(s_1,s_2,\ldots,s_{39},0),
$$

且最后一个坐标为 $a/p$。先对基做 LLL 约化，再用 Babai 最近平面法求近似最近向量，便可从最后一维还原 $a$。

完整 Sage 脚本如下：

```python
import time
from ast import literal_eval

from Crypto.Util.number import long_to_bytes
from sage.all import Matrix, QQ, ZZ, next_prime, vector

d = 39
p = next_prime(2**360)

with open("output.txt", "r", encoding="utf-8") as stream:
    outputs = literal_eval(stream.read())

inputs = outputs[:d]
answers = outputs[1:d + 1]
assert len(inputs) == len(answers) == d


def build_basis(oracle_inputs):
    basis_vectors = []
    for i in range(d):
        row = [0] * (d + 1)
        row[i] = p
        basis_vectors.append(row)

    basis_vectors.append(list(oracle_inputs) + [QQ(1) / QQ(p)])
    return Matrix(QQ, basis_vectors)


def approximate_closest_vector(basis, target):
    reduced = basis.LLL()
    gram_schmidt, _ = reduced.gram_schmidt()
    _, dimension = reduced.dimensions()

    residual = vector(ZZ, target)
    for i in reversed(range(dimension)):
        coefficient = QQ(residual * gram_schmidt[i]) / QQ(
            gram_schmidt[i] * gram_schmidt[i]
        )
        residual -= reduced[i] * coefficient.round()

    return (target - residual).coefficients()


started = time.time()
lattice = build_basis(inputs)
target = vector(ZZ, list(answers) + [0])
closest = approximate_closest_vector(lattice, target)

recovered_a = ZZ((closest[-1] * p) % p)
print(long_to_bytes(recovered_a))
print(f"elapsed: {time.time() - started:.3f}s")
```

输出为：

```text
b'hgame{Repair_modulus_prob1em_5o_HNP_Revenge}'
```

官方 PDF 给出了同一套 LLL/Babai 实现，但没有记录最终 flag；结果由参赛者的 [HGAME 2023 Week 4 题解](https://lazzzaro.github.io/2023/02/06/match-HGAME-2023-Week-4/index.html) 补全。

## 方法总结

带小误差的模线性样本是典型 HNP。建格前应先量化信息差：本题模数约 360 位、误差最多 340 位，每个样本泄露约 20 位高位信息。缩放用的最后一维 `1/p` 很关键，它让未知数 $a$ 能作为短距离的一部分被 CVP 恢复；LLL 负责改善基，Babai 则把目标向量投影到近似最近格点。
