# DownUnderCTF 2023 hhhhh Writeup

## 题目简述

程序对输入的每个前缀分别计算 MD5，并把所有前缀摘要逐字节异或。目标是构造输入，使最终 16 字节结果全部等于 ASCII 字符 `h`。直接求原像困难，但 MD5 碰撞可以把长输入拆成许多相互独立的二元选择。

## 解题过程

定义聚合函数：

$$
H(m)=\bigoplus_{i=1}^{|m|}\operatorname{MD5}(m[0:i]).
$$

先取固定前缀 `a` 重复 128 次。对当前前缀使用 MD5 碰撞生成器得到两个 128 字节块 $A_i,B_i$，使：

$$
\operatorname{MD5}(P_i\parallel A_i)
=\operatorname{MD5}(P_i\parallel B_i).
$$

碰撞块结束时的 MD5 内部状态相同，因此之后追加相同后缀时不会受本次选择影响；差异只存在于该 128 字节块内部各前缀摘要的异或和。连续生成 128 对块，就得到 128 个独立的 128 位差分向量。

仓库已经提供预计算的 `f2a.bin` 和 `f2b.bin`。将全部选择默认设为 $B_i$，再令矩阵第 $i$ 列为 $H_i(A_i)\oplus H_i(B_i)$，在 $\operatorname{GF}(2)$ 上求解即可：

```python
from hashlib import md5
from sage.all import GF, Matrix, vector

def xor_bytes(left, right):
    return bytes(a ^ b for a, b in zip(left, right))

def effect(prefix, block):
    result = bytes(16)
    combined = prefix + block
    for end in range(len(prefix), len(combined)):
        result = xor_bytes(result, md5(combined[:end + 1]).digest())
    return int.from_bytes(result, "big")

prefix = b"a" * 128
raw_a = open("f2a.bin", "rb").read()
raw_b = open("f2b.bin", "rb").read()
blocks = [
    (raw_a[i:i + 128], raw_b[i:i + 128])
    for i in range(0, len(raw_a), 128)
]

columns = []
baseline = 0
current = prefix
for block_a, block_b in blocks:
    value_a = effect(current, block_a)
    value_b = effect(current, block_b)
    columns.append(value_a ^ value_b)
    baseline ^= value_b
    current += block_a

prefix_value = bytes(16)
for end in range(len(prefix)):
    prefix_value = xor_bytes(prefix_value, md5(prefix[:end + 1]).digest())
baseline ^= int.from_bytes(prefix_value, "big")

field = GF(2)
matrix = Matrix(
    field,
    128,
    128,
    lambda row, col: (columns[col] >> (127 - row)) & 1,
)
target = int.from_bytes(b"h" * 16, "big") ^ baseline
rhs = vector(field, [(target >> (127 - row)) & 1 for row in range(128)])
choice = matrix.solve_right(rhs)

answer = prefix
for bit, (block_a, block_b) in zip(choice, blocks):
    answer += block_a if bit else block_b
print(answer.hex())
```

提交构造出的长十六进制串，服务返回：

```text
DUCTF{hhh.hhh_hh_hhhhhh_hhh,hh_hh?h_hh_hhhh_hhh-hh,hhh-hhhhh_hhhhh_hh.}
```

## 方法总结

MD5 碰撞本身不能直接给出任意原像，但可把每一对碰撞块变成“选 A 或选 B”的独立开关。目标聚合函数又是按位异或，于是 128 个开关恰好形成 128 维线性系统。利用碰撞的可延展性与外层 XOR 的线性，原像问题就转化为一次高斯消元。
