# i luv linear

## 题目简述

题目把 32 字节 flag 视为 256 位整数，并在固定随机种子下重复 100 次 `ct ^= ct >> k`。虽然代码看起来在不断扰动比特，但所有操作都在 $\mathrm{GF}(2)$ 上线性，目标是建立整体线性变换并求逆。

## 解题过程

异或是 $\mathrm{GF}(2)$ 上的加法，右移只是重新排列并丢弃部分坐标，因此每一步 $x\mapsto x\oplus(x\gg k)$ 都是线性映射。固定 `random.seed(0)` 后，100 个移位量完全确定，整个加密可表示为：

$$c=xM$$

构造矩阵时，依次对 256 个基向量 $2^i$ 执行同一加密，将输出比特向量作为矩阵的行。随后在 $\mathrm{GF}(2)$ 上求逆：

```python
import random

def transform(x):
    random.seed(0)
    for _ in range(100):
        x ^= x >> random.randint(1, 32)
    return x

def bits(x, n=256):
    return vector(GF(2), [(x >> i) & 1 for i in range(n)])

M = Matrix([bits(transform(1 << i)) for i in range(256)])
plain_bits = bits(ciphertext) * M.inverse()
```

把求得的 256 位向量按原位序还原为整数与 32 字节串，得到：

```text
grey{m4tr1ces_4re_s0_c00l_heheh}
```

## 方法总结

异或、移位、比特置换常会组成线性变换。只要随机参数可复现，就能通过基向量查询构造完整矩阵，无须逐步猜测逆操作；矩阵可逆时直接在线性域上求逆即可。
