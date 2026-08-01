# GlacierCTF2023 - SLCG

## 题目简述

加密器维护两个独立 LCG。每个 ASCII 字符按 7 位二进制展开：比特为 0 时输出零号 LCG 的下一个值，比特为 1 时输出一号 LCG 的下一个值。每处理完一个字符，又各取旧 LCG 的四个后继值作为新 LCG 的模数、乘数、加数和种子。

## 解题过程

flag 已知以 `g` 开头，而 `g` 的 7 位表示为 `1100111`。因此密文位置 0、1、4、5、6 是一号 LCG 的五个连续输出。设连续值为 $x_i$，差分 $d_i=x_{i+1}-x_i$，则

$$
d_i d_{i+2}-d_{i+1}^2\equiv0\pmod m.
$$

对多组该表达式取最大公因数可恢复模数 $m$；之后计算

$$
a=(x_2-x_3)(x_1-x_2)^{-1}\bmod m,
\qquad
c=x_2-a x_1\bmod m.
$$

从第一个输出开始跟踪一号 LCG。若预测的下一个值等于当前密文值，该位为 1 并提交状态；否则该位为 0，回滚一号 LCG 的状态，因为加密端此时推进的是另一个 LCG。每七位结束后，再从已知 LCG 连取四个值构造下一代：

```python
for encrypted_char in chunks(ciphertext, 7):
    for value in encrypted_char:
        old = lcg.value
        if next(lcg) == value:
            bits.append(1)
        else:
            bits.append(0)
            lcg.value = old

    lcg = LCG(next(lcg), next(lcg), next(lcg), next(lcg))
```

把恢复的每七位转回字符，得到：

```text
gctf{th15_lcg_3ncryp710n_w4sn7_s0_5s3cur3_aft3r_4ll}
```

## 方法总结

这里泄漏的不是“随机数偏置”，而是同一 LCG 的连续输出和明文选择之间的直接对应。已知格式给出了足够长的同源输出序列；恢复一个分支后，逐位预测还能持续跟踪跨字符的递归参数更新。
