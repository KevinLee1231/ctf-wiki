# GreyCTF2022 - Logical Computers

## 题目简述

附件是一个 PyTorch 两层网络的权重。模型并非做自然数据分类，而是把 flag 的每一位作为布尔输入；隐藏层神经元编码大量 AND 子句，输出层要求所有子句同时成立。需要从权重恢复布尔约束。

## 解题过程

读取 `state_dict`，检查第一层权重和偏置。对二值输入，正权重表示要求该位为 1，负权重配合偏置表示要求该位为 0；每个隐藏神经元仅在其整条子句满足时激活。把每条子句翻译为 Z3 条件：

```python
bits = [Bool(f'b{i}') for i in range(input_size)]
clauses = []
for weight, bias in hidden_neurons:
    terms = []
    for i, w in enumerate(weight):
        if w > 0:
            terms.append(bits[i])
        elif w < 0:
            terms.append(Not(bits[i]))
    clauses.append(And(*terms))

solver.add(And(*clauses))
```

求解后按模型输入约定每 8 位组成一个字符。源码使用低位在前，故第 $j$ 位需放到 $1\ll j$：

```text
grey{sM0rT_mAch1nE5}
```

## 方法总结

模型题要先判断神经网络是否只是逻辑电路的数值表示。本题无需训练，也不应靠梯度猜输入；权重的离散取值和偏置直接给出子句。还原后必须把 bit vector 再送入原模型，确认输出确实落在接受区域。
