# C O N N E C T 1 0 N

## 题目简述

题目把 flag 分成 4 段，每段 padding 到 128 bit。源码中 `shuffle(left,right)` 生成由固定数量 0/1 随机排列出的 128 bit 掩码，再与 flag 段异或，形成二层数据矩阵：

```sage
l, r = 63, 65
S = [bytes_to_long(padding(flag_part, (l + r) // 8)) for flag_part in parts]
data = [
  [
    [
      shuffle(r, l) ^^ S[_] if j == "1" else shuffle(l, r) ^^ S[_]
      for j in (...)
    ]
    for i in bin(S[_])[2:].rjust(l + r, "0")
  ]
  for _ in range(4)
]
```

`padding()` 会保证整体 bit 中 1 的个数为奇数。

## 解题过程

### 关键机制

把 bit 映射为矩阵元素：$0\to1$，$1\to-1$。若只有一层循环，则可得到：

$$
\text{flag}\cdot M=-2\text{flag}
$$

即：

$$
\text{flag}\cdot(M+2I)=0
$$

可通过左核恢复 flag。

题目是两层循环，上述关系扩展为：

$$
\text{flag}\cdot\sum_i M_i=4\text{flag}
$$

因此：

$$
\text{flag}\cdot\left(-4I+\sum_iM_i\right)=0
$$

又因为源码保证 1 的个数为奇数，矩阵关系在模 8 下有结构，可用格规约恢复 $\pm1$ 向量。

### 求解步骤

对每个 flag 段分别处理。读取 `output.txt` 后，把每个整数转成 128 bit，并把 bit 转成 $\pm1$：

```python
from Crypto.Util.number import *

data = eval(open("output.txt").read())
for part in range(4):
    M = matrix(128, 128)
    for i in range(128):
        for j in range(128):
            temp = bin(data[part][i][j])[2:].zfill(128)
            row = [-1 if b == "1" else 1 for b in temp]
            for k in range(128):
                M[k, i] += row[k]
```

构造格：

```python
T = block_matrix([
    [identity_matrix(128), M],
    [0, identity_matrix(128) * 8]
])
T[:, -128:] *= 2**10
res = T.BKZ(block_size=30)
```

寻找前 128 维全为 $\pm1$ 的短向量，将两种符号分别转成 bit 串：

```python
for v in res:
    if all(abs(x) == 1 for x in v[:128]):
        ans1 = "".join("1" if x == -1 else "0" for x in v[:128])
        ans2 = "".join("0" if x == -1 else "1" for x in v[:128])
        print(long_to_bytes(int(ans1, 2)))
        print(long_to_bytes(int(ans2, 2)))
```

四段拼接后去除随机 padding 即可恢复 flag。

## 方法总结

- 题目核心是 XOR 的线性/同态关系，而不是暴力枚举 padding。
- 映射 $0\to1,1\to-1$ 后，随机 shuffle 的计数差异会转成矩阵核问题。
- 两层循环让关系变成 $-4I+\sum_iM_i$，再借助模 8 结构用 BKZ 找 $\pm1$ 短向量。
