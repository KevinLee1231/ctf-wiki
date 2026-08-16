# HackINI2024 BNNs

## 题目简述

题目提供一个输入维度和输出维度均为 50、中间层为 512 的全连接神经网络，以及被篡改的模型权重和输入数据。模型参数中被注入了很大的数值，正常推理会得到数量级异常的输出；需要根据官方求解思路修复权重，再让模型对指定句子输出 flag。

## 解题过程

首先不要直接以默认方式反序列化来历不明的 `.pt` 或 Pickle 文件。权重可使用 `weights_only=True` 加载；输入数据则应使用限制全局对象的安全 Unpickler，或在隔离环境中检查。安全读取后可确认：

- 输入列表共有 332 项，第 69 项是 `Write the flag`；
- 权重字典的键名与题目中的网络结构一致；
- 参数绝对值最大约为 `900000.3125`，其中 508 个参数的绝对值不小于 100；
- 未修复时，输出约落在 $-1.89\times 10^{13}$ 至 $6.36\times 10^{13}$，显然不是字符编码。

篡改方式是在正常参数上叠加 100 的大倍数，因此对每个参数执行浮点取模即可保留原始的小数部分：

```python
import pickle

import torch
import torch.nn as nn


class NoGlobalsUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        raise pickle.UnpicklingError(
            f"global object is forbidden: {module}.{name}"
        )


class MyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.model = nn.Sequential(
            nn.Linear(50, 512),
            nn.ReLU(),
            nn.Linear(512, 50),
        )

    def forward(self, x):
        return self.model(x)


with open("data", "rb") as file:
    data = NoGlobalsUnpickler(file).load()

model = MyModel()
state = torch.load(
    "hacked_model.pt",
    map_location="cpu",
    weights_only=True,
)
model.load_state_dict(state)

with torch.no_grad():
    for parameter in model.parameters():
        parameter.fmod_(100)

sentence = data[69]
tokens = torch.tensor([ord(char) for char in sentence], dtype=torch.float32)
sample = torch.zeros(50)
sample[:len(tokens)] = tokens

with torch.no_grad():
    result = model(sample)

chars = result.round().clamp_min(0).to(torch.int64).tolist()
text = "".join(map(chr, chars)).rstrip("\x00")
print(text)
```

修复后参数绝对值最大约为 `5.45`，输出取整并去掉末尾的零填充后得到：

```text
shellmates{expert_in_debuggin}
```

## 方法总结

本题的决定性线索是权重中出现了与正常神经网络参数尺度不相称的大数，而模 100 能稳定消除注入的整百倍偏移。处理模型附件时还要把反序列化风险视为解题的一部分：优先使用 `weights_only=True`、受限 Unpickler 和 CPU 映射，确认结构后再推理。最后通过输出范围、ASCII 合法性和 flag 格式共同验证修复结果。
