# A very happy MLP

## 题目简述

题目给出一个三层全连接网络、训练好的参数和基准输入 $x$，要求构造一个张量 $f$，使网络满足：

$$
\operatorname{Net}(x+f)=(2,3,3).
$$

源码中的网络维度依次为 $30\to20\to10\to3$。前两层在线性变换后使用 Sigmoid，并把 $(0,1)$ 仿射放大到 $(-20,20)$；最后一层不带激活。因此本题不是训练或梯度优化，而是按层逆推一个满足目标输出的输入原像。

## 解题过程

按列向量记法，一层全连接为：

$$
y=Wx+b.
$$

矩阵 $W$ 不是方阵，不能直接求逆，但可以使用 Moore-Penrose 伪逆取一个最小范数解：

$$
x=W^+(y-b).
$$

PyTorch 的 `Linear` 权重形状为“输出维度 $\times$ 输入维度”，代码采用行向量表示，因此实现为：

```python
def inv_fc(y, weight, bias):
    return (y - bias) @ torch.linalg.pinv(weight).T
```

前两层的激活不是单纯的 Sigmoid，而是：

$$
a=40\sigma(z)-20.
$$

先还原 $s=(a+20)/40$，再使用 logit：

$$
z=\ln\frac{s}{1-s}.
$$

只有 $0<s<1$ 时逆变换才有定义；也就是待逆的中间值必须位于 $(-20,20)$。本题参数经过构造，伪逆得到的原像满足这一条件。若迁移到任意网络后越界，则不能机械套用最小范数解，需要在伪逆解的零空间中继续寻找可行原像。

完整逆推顺序与前向传播相反：先逆 `fc3`，再逆第二个缩放 Sigmoid 和 `fc2`，最后逆第一个缩放 Sigmoid 和 `fc1`。得到网络输入后减去题目给出的 `base_input`，就是需要提交的 flag 张量。

```python
import numpy as np
import torch

from model import Net


def inv_fc(y, weight, bias):
    return (y - bias) @ torch.linalg.pinv(weight).T


def inv_scaled_sigmoid(a, scale=40.0):
    s = (a + scale / 2) / scale
    if not torch.all((0 < s) & (s < 1)):
        raise ValueError("逆 Sigmoid 的输入越出有效范围")
    return torch.log(s / (1 - s))


checkpoint = torch.load("_Checkpoint.pth", map_location="cpu")
net = Net()
net.load_state_dict(checkpoint["model"])
net.eval()

target = torch.tensor([2.0, 3.0, 3.0])

a2 = inv_fc(target, net.fc3.weight, net.fc3.bias)
z2 = inv_scaled_sigmoid(a2, net.scale)
a1 = inv_fc(z2, net.fc2.weight, net.fc2.bias)
z1 = inv_scaled_sigmoid(a1, net.scale)
required_input = inv_fc(z1, net.fc1.weight, net.fc1.bias)

flag_tensor = required_input - checkpoint["input"]
torch.save(flag_tensor.detach(), "_ExpFlag.pth")

with torch.no_grad():
    output = net(checkpoint["input"] + flag_tensor)
assert torch.equal(torch.round(output, decimals=5), target)

flag_body = "".join(
    chr(round((float(value) + 1) / 2 * 255))
    for value in flag_tensor.flatten()
)
print(f"moectf{{{flag_body}}}")
```

最终输出为题目内置的 flag；提交文件是 `_ExpFlag.pth`，而不是打印出的字符串。

## 方法总结

本题的关键是区分“求一个输入原像”和“训练模型”。线性层降维后没有唯一逆，但伪逆能给出一个可验证的原像；单调激活可以在值域内逐点求逆。逆推神经网络时应从输出层向输入层逐层进行，并在每一步检查激活函数值域，最后用原模型重新前向验证结果。
