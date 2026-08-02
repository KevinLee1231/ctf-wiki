# ml-project

## 题目简述

题目给出 PyTorch 模型参数 `model.pth`、对未知 flag 的输出 `output.pth` 和编码代码。网络结构是 26 维输入、22 维隐藏层、18 维输出层：

$$
y=W_2\operatorname{ReLU}(W_1x+b_1)+b_2.
$$

输入 $x$ 是 flag 各字符的 ASCII 值。所有权重和偏置都是 0 到 9 的整数，输入也为非负数，所以第一层每个值均非负，ReLU 实际上是恒等映射。整个模型退化为整数线性方程组。

## 解题过程

加载 `state_dict` 和输出，转为整数后，把 26 个 ASCII 字符声明为 Z3 整数。加入 `tjctf{` 前缀、结尾 `}` 和 ASCII 范围约束，再按两层线性运算建立等式：

```python
import torch
import z3

FLAG_LEN = 26

class Model(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.f1 = torch.nn.Linear(FLAG_LEN, 22, dtype=torch.float64)
        self.f2 = torch.nn.Linear(22, 18, dtype=torch.float64)

model = Model()
model.load_state_dict(torch.load("model.pth", weights_only=True))
target = torch.load("output.pth", weights_only=True)[0].round().to(torch.int64)

W1 = model.f1.weight.round().to(torch.int64).tolist()
b1 = model.f1.bias.round().to(torch.int64).tolist()
W2 = model.f2.weight.round().to(torch.int64).tolist()
b2 = model.f2.bias.round().to(torch.int64).tolist()

x = [z3.Int(f"x_{i}") for i in range(FLAG_LEN)]
solver = z3.Solver()
for i, char in enumerate(b"tjctf{"):
    solver.add(x[i] == char)
for value in x:
    solver.add(0 <= value, value < 128)
solver.add(x[-1] == ord("}"))

hidden = [z3.Sum([W1[r][c] * x[c] for c in range(FLAG_LEN)]) + b1[r]
          for r in range(22)]
for r in range(18):
    value = z3.Sum([W2[r][c] * hidden[c] for c in range(22)]) + b2[r]
    solver.add(value == int(target[r]))

assert solver.check() == z3.sat
answer = solver.model()
print("".join(chr(answer.eval(v).as_long()) for v in x))
```

求解结果为：

```text
tjctf{1m_4_r0b0t_75dff3c2}
```

## 方法总结

- 是否属于 AI/ML 题取决于决定性机制：这里必须读取模型结构与权重，但网络最终退化为线性约束。
- ReLU 不能无条件删除；本题之所以能删，是因为权重、偏置和 ASCII 输入全非负，应在正文明确证明。
- 浮点输出来自小整数的有限次乘加，使用 float64 足以精确表示这些值；转入 Z3 前仍按官方脚本四舍五入，避免序列化显示误差。
