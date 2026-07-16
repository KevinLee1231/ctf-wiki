# Take my bag

## 题目简述

题目把明文整数的二进制位反转后，按置位位置累加 $w\cdot3^i\bmod n$，属于结构非常弱的背包加密。已知 $w$ 和 $n$，先消去乘数 $w$，剩余权重 $1,3,9,\ldots$ 构成超递增序列，可用贪心算法恢复每一位。

## 解题过程

加密循环等价于：

$$
c\equiv w\sum_i b_i3^i\pmod n,\qquad b_i\in\{0,1\}.
$$

由于 $w$ 是 64 位素数且与 512 位素数 $n$ 互素，可以计算 $w^{-1}\bmod n$：

$$
s=cw^{-1}\bmod n=\sum_i b_i3^i.
$$

对本题给定明文，实际的幂次范围使右侧总和小于 $n$，因此最后一次模约减没有破坏这个整数和。又因为 $3^i>\sum_{j=0}^{i-1}3^j$，从最大幂开始依次判断能否从 $s$ 中减去对应权重，就能唯一恢复所有 $b_i$。

```python
from Crypto.Util.number import inverse, long_to_bytes

w = 16221818045491479713
n = ...  # 题目给出的模数
c = ...  # 题目给出的密文

s = c * inverse(w, n) % n

highest = 0
while 3 ** highest <= s:
    highest += 1

m = 0
for i in range(highest - 1, -1, -1):
    weight = 3 ** i
    if weight <= s:
        s -= weight
        m |= 1 << i

assert s == 0
print(long_to_bytes(m))
```

源码本来就是从明文最低位开始对应 $3^0$，所以恢复出的置位位置可直接写回整数，不需要再次反转字符串。

## 方法总结

解背包题应先把代码写成数学式，再检查权重是否超递增、是否经过置换或模乘，以及模运算是否真的发生了回绕。本题的安全性问题是去掉可逆乘数后仍暴露了简单的三进制超递增结构；“总和小于模数”只对本题实际明文成立，不能误写成对所有 512 个权重都成立。
