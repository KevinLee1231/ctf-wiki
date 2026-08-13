# GreyCTF2022 - QFC

## 题目简述

题目提供一个模拟量子 oracle，可按索引查询 flag 字符。对字符的每一位，电路先制备 $|+\rangle$，再由秘密位决定是否施加 $Z$ 相位，最后再次施加 Hadamard；测量结果就是该秘密位。

## 解题过程

单个量子位的状态变化为：

$$H|0\rangle=|+\rangle,$$

秘密位为 0 时不变，最终 $H|+\rangle=|0\rangle$；秘密位为 1 时施加 $Z$ 得到 $|-\rangle$，最终 $H|-\rangle=|1\rangle$。因此对每个字符索引运行服务给出的电路，统计足够多次测量的众数即可还原 8 位。

```python
value = 0
for bit in range(8):
    counts = run_circuit(index, bit, shots=5000)
    measured = max(counts, key=counts.get)
    value |= int(measured) << bit
chars.append(value)
```

按低位在前组合字符，依次查询到右花括号，得到：

```text
grey{Qu4nTuM_I5_S0_sC4Ry}
```

## 方法总结

这题使用的是确定性的相位回踢/基变换关系，5000 次 shots 只是让模拟接口的统计结果更显著。应先在纸面推导 $H Z^b H|0\rangle=|b\rangle$，再处理字符位序；不需要把它误解为真实硬件噪声分析。
