# Level 314 线性走廊中的双生实体

## 题目简述

附件是一个 TorchScript 模型 `model.pt`。模型接收形状为 $(1,10)$ 的张量，正常执行时会打印经凯撒位移还原的假 flag；只有输入均值满足隐藏的 `torch.allclose` 条件时，才会把另一组常量按 `0x55` 异或并打印真 flag。

题名中的 “314” 与提示诗句里的“周率”“十方”共同指向 $pi/10\approx0.31415$。决定解法的是模型内部控制流与触发输入，因此归入 `ai-ml`。

## 解题过程

TorchScript 会保留可执行的前向逻辑。加载模型并查看 `model.code`，可以看到关键判断等价于：

```python
if torch.allclose(
    torch.mean(x),
    torch.tensor(0.31415000000000004),
    rtol=1.0000000000000001e-05,
    atol=0.0001,
):
    # 将 real_flag 中的每个字节与 0x55 异或后打印
    ...
```

模型生成时的核心逻辑如下。假 flag 使用字符码加 $3$ 保存，真 flag 则按字节异或 `0x55` 保存：

```python
class MyModule(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = nn.Linear(10, 1)
        self.real_flag = [x ^ 0x55 for x in b"flag{s0_th1s_1s_r3al_s3cr3t}"]
        self.fake_flag = [ord(c) + 3 for c in "flag{fake_flag}"]

    def forward(self, x):
        fake = "".join(chr(c - 3) for c in self.fake_flag)
        print("Fake flag:", fake)

        if torch.allclose(
            torch.mean(x),
            torch.tensor(3.1415 / 10),
            atol=1e-4,
        ):
            real = "".join(chr(b ^ 0x55) for b in self.real_flag)
            print("Real flag:", real)

        return self.linear(x)
```

因此不需要修改模型，也不需要枚举输入。构造十个元素均为 $0.31415$ 的张量即可稳定满足均值条件：

```python
import torch

model = torch.jit.load("model.pt")
print(model.code)

trigger = torch.full((1, 10), 3.1415 / 10)
model(trigger)
```

执行后模型先输出假 flag，随后进入隐藏分支并输出：

```text
Real flag: flag{s0_th1s_1s_r3al_s3cr3t}
```

## 方法总结

- 核心技巧：检查 TorchScript 的 `forward` 控制流和序列化常量，直接构造满足隐藏统计条件的模型输入。
- 识别信号：模型固定打印诱饵结果，同时题面强调输入形状、稳定态、均值或某个特殊常数时，应优先检查数据依赖分支，而不是把模型只当黑盒推理器。
- 复用要点：`torch.allclose` 同时受目标值、相对误差和绝对误差约束；直接填充目标常数比随机搜索更可靠。分析模型时既要查看参数，也要查看脚本化控制流和非参数常量。
