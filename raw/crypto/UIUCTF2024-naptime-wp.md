# Naptime

## 题目简述

附件实现了 Merkle--Hellman 背包加密的公开侧：每个明文字节被拆成 8 个比特，并计算 $t=\sum_{i=0}^{7}b_i a_i$。公开文件给出 8 个背包权重和每个字符对应的子集和。目标是从每个 $t$ 恢复只取 $0$ 或 $1$ 的比特向量。

## 解题过程

对单个密文 $t$，构造九维格基：前 8 行分别是单位向量 $e_i$，并在最后一列放入 $a_i$；最后一行只有末项 $-t$。若正确比特为 $(b_0,\ldots,b_7)$，把前 8 行按 $b_i$ 线性组合，再加一次最后一行，就得到

$$
(b_0,b_1,\ldots,b_7,\sum b_i a_i-t)
=(b_0,b_1,\ldots,b_7,0).
$$

这个向量各分量只有 0 或 1，欧氏长度很短，LLL 约化后通常会直接出现在基中。逐个密文建立格并筛选候选即可：

```python
from sage.all import Matrix, ZZ

def recover_byte(target, weights):
    basis = Matrix(ZZ, 9, 9)
    for index, weight in enumerate(weights):
        basis[index, index] = 1
        basis[index, 8] = weight
    basis[8, 8] = -target

    for reduced_row in basis.LLL().rows():
        for row in (list(reduced_row), list(-reduced_row)):
            bits = row[:8]
            if row[8] != 0 or any(bit not in (0, 1) for bit in bits):
                continue
            if sum(bit * weight for bit, weight in zip(bits, weights)) == target:
                return int("".join(map(str, bits)), 2)
    raise ValueError("no subset-sum solution")

weights = [66128, 61158, 36912, 65196, 15611, 45292, 84119, 65338]
plaintext = bytes(recover_byte(target, weights) for target in ciphertext)
print(plaintext.decode())
```

对 `pub.txt` 中全部 39 个子集和运行后得到：

```text
uiuctf{i_g0t_sleepy_s0_I_13f7_th3_fl4g}
```

本题每组实际上只有 8 个权重，枚举 $2^8=256$ 个子集也足以恢复一个字节；LLL 解法则展示了当维数增大时更通用的低密度背包攻击思路。

## 方法总结

- 把子集和等式嵌入格的最后一维，正确的 0/1 解会对应末项为 0 的异常短向量。
- LLL 只负责给出短候选，仍应检查前 8 项是否为比特、最后一项是否为 0，并重新计算子集和，避免误收其他短向量。
- 选择方法时也要看实际参数：8 位背包可直接枚举，而格攻击说明了这类加密结构在更一般参数下为何不安全。
