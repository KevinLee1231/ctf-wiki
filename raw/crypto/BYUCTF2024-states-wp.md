# States

## 题目简述

题面给出一个整数关于 31、23、47 的三个余数，并说明结果最终对应美国州的两字母缩写。三个模数两两互素，因此先用中国剩余定理求唯一同余类，再把结果转成 36 进制字母。

## 解题过程

同余组为

$$
x\equiv15\pmod{31},\quad
x\equiv19\pmod{23},\quad
x\equiv11\pmod{47}.
$$

令 $N=31\cdot23\cdot47$，$N_i=N/n_i$，再计算 $N_i$ 在模 $n_i$ 下的逆元，便有

$$
x=\sum_i a_iN_i(N_i^{-1}\bmod n_i)\pmod N.
$$

Python 可直接实现：

```python
from functools import reduce

mods = [31, 23, 47]
rems = [15, 19, 11]
N = reduce(int.__mul__, mods)
x = sum(a * (N // n) * pow(N // n, -1, n)
        for a, n in zip(rems, mods)) % N
print(x)  # 387
```

$387=10\times36+27$，按 `0-9A-Z` 的 36 进制字母表对应 `AR`，即 Arkansas：

```text
byuctf{arkansas}
```

## 方法总结

先用 CRT 合并同余条件，再依据题面语义解释结果。数值 `387` 不是终点；“两字母州缩写”提示了 36 进制表示，并将 `AR` 唯一映射到 Arkansas。
