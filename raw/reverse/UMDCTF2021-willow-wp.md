# Willow

## 题目简述

题目是裸机 ARM 程序。输入校验使用一棵 31 节点二叉树把 4 位 nibble 映射为新值，再通过前一输出字节形成链式关系；树映射还不是严格双射，需要用可打印字符和 flag 格式消除歧义。

## 解题过程

静态数据中：

- 目标密文共 30 字节，位于约 `0x868`；
- 31 节点树位于约 `0x7ec`；
- 每个输入字节拆成高、低 nibble，按特定位序沿树左右分支，到叶节点取得替换值。

先按程序实际读取位的顺序重建替换函数 $S$。校验关系可整理为：

$$
y_0=\neg S(x_0),
$$

$$
y_i=S(x_i)\oplus y_{i-1}\quad(i>0).
$$

因此对每个位置先由目标 $y_i$ 求出所需的 $S(x_i)$，再枚举所有能映射到该值的输入字节：

```python
needed = (~target[0]) & 0xff
candidates0 = inverse_s[needed]

for i in range(1, len(target)):
    needed = target[i] ^ target[i - 1]
    candidates_i = inverse_s[needed]
```

由于树映射存在碰撞，`inverse_s` 的某些项不止一个候选。使用 `UMDCTF-{` 前缀、末尾 `}` 和可打印 ASCII 约束选择完整字符串：

```text
UMDCTF-{WellthatWasSomething?}
```

## 方法总结

自定义树替换的重点不是给节点强行命名成 S 盒，而是忠实还原每一位决定的分支和叶值。映射非双射时，逆运算天然产生候选集合；链式异或提供逐位置约束，flag 格式负责最后消歧，不能随意选第一个逆像。
