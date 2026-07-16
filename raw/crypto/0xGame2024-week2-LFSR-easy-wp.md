# LFSR-easy

## 题目简述

题目给出 128 位 LFSR 连续产生的两段 128 位输出，未知量是 128 位反馈掩码。每一位输出都是当前状态与掩码在 $\mathrm{GF}(2)$ 上的内积，因此 128 个连续状态可以组成线性方程组，直接求出掩码。

## 解题过程

状态更新为：

$$
o_t=\sum_{i=0}^{127}s_{t,i}m_i\pmod2
$$

$$
S_{t+1}=(s_{t,1},s_{t,2},\ldots,s_{t,127},o_t)
$$

第一段 128 位输出结束后，寄存器状态恰好就是这 128 个输出位。把两段输出依次记为 $O_0,O_1$，则从 $O_0$ 开始的 128 个滑动窗口就是已知状态矩阵，每行对应的输出是 $O_1$ 的一位：

$$
A M=b\quad\text{over }\mathrm{GF}(2)
$$

题目给出的两段输出为：

```text
O0 = 299913606793279087601607783679841106505
O1 = 192457791072277356149547266972735354901
```

使用 SageMath 求解：

```python
from hashlib import md5

O0 = 299913606793279087601607783679841106505
O1 = 192457791072277356149547266972735354901

o0 = [int(bit) for bit in f"{O0:0128b}"]
o1 = [int(bit) for bit in f"{O1:0128b}"]
sequence = o0 + o1

A = Matrix(GF(2), [
    sequence[index:index + 128]
    for index in range(128)
])
b = vector(GF(2), o1)

mask = A.solve_right(b)
mask_bits = "".join(str(int(bit)) for bit in mask)
mask_seed = int(mask_bits, 2)

print(mask_seed)
print("0xGame{" + md5(str(mask_seed).encode()).hexdigest() + "}")
```

结果为：

```text
267531662175765080829888561132601949966
0xGame{d56821feacab64cdb87c754ad06823a2}
```

## 方法总结

LFSR 的“异或与移位”在 $\mathrm{GF}(2)$ 上是线性的。只要能获得足够多的连续输出，就可以把未知反馈掩码视为线性方程组的变量；本题两段各 128 位的输出正好提供了构造状态矩阵和右端向量所需的数据。
