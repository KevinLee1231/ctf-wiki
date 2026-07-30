# SU_AI_call_white_black

## 题目简述

题目构造了一个基于 CIFAR-10 和 ResNet-18 的联邦学习场景。服务器已有全局模型，并把十个客户端的参数更新做简单平均。攻击者控制其中一个客户端，需要上传一份更新，使聚合后的模型同时满足：

- 正常测试集准确率高于 $70\%$；
- 带后门触发器的测试集被预测为类别 `1` 的成功率高于 $95\%$。

触发器是在图像第 `0` 通道的第 `2` 至 `27` 行、第 `3` 至 `5` 列写入 `1.0`，并把另外两个通道同一区域置零，即一条红色竖线。漏洞不在 HTTP 上传接口，而在无裁剪、无异常检测的线性模型聚合。

当前总仓库只保留了很短的官方思路，没有模型和训练数据。聚合代码、评测门槛、训练日志与最终结果由[公开的完整赛后题解](https://outw.rest/posts/suctf2025-ai-writeups)补足；下面已把其中影响解法的内容完整写入正文。

## 解题过程

### 1. 训练目标后门模型

先从服务器给出的初始全局模型 $O$ 继续训练。每个 batch 混入一部分带红线触发器的图像，并把这些图像的标签改成 `1`；其余样本保持原图和原标签。训练目标是得到模型 $T$：

- 对正常图像仍保持较高准确率；
- 对任意带红线的图像几乎都输出类别 `1`。

核心数据投毒逻辑如下：

```python
trigger_pos = [(row, col) for row in range(2, 28) for col in range(3, 6)]

for images, labels in train_loader:
    poison_count = max(1, images.size(0) // 10)
    for index in range(poison_count):
        for row, col in trigger_pos:
            images[index, 0, row, col] = 1.0
            images[index, 1, row, col] = 0.0
            images[index, 2, row, col] = 0.0
        labels[index] = 1

    optimizer.zero_grad()
    loss = criterion(model(images.cuda()), labels.cuda())
    loss.backward()
    optimizer.step()
```

公开复现使用 `AdamW(lr=0.001)` 训练十轮，最终正常验证准确率约为 $80.58\%$，后门成功率约为 $99.98\%$。这说明 $T$ 已同时满足两类行为要求。

### 2. 反解要上传的恶意更新

设服务器初始模型参数为 $O$，十个客户端上传的参数差分别为 $\Delta_1,\ldots,\Delta_{10}$。服务器的聚合结果为

$$
F=O+\frac{1}{10}\sum_{i=1}^{10}\Delta_i.
$$

攻击者希望 $F=T$。若能估计九个正常客户端的更新，则恶意客户端应提交

$$
\Delta_{\mathrm{mal}}
=10(T-O)-\sum_{i=1}^{9}\Delta_i.
$$

官方思路指出，训练到后期时正常客户端相对全局模型的差已经很小，可以近似忽略，于是有

$$
\Delta_{\mathrm{mal}}\approx 10(T-O).
$$

若上传格式是“本地模型参数”而不是参数差，则要构造

$$
W_{\mathrm{mal}}=O+\Delta_{\mathrm{mal}}
\approx O+10(T-O).
$$

对应代码是：

```python
target_state = target_backdoor_model.state_dict()
global_state = global_model.state_dict()
malicious_state = {}

for name in target_state:
    malicious_state[name] = (
        global_state[name]
        + 10 * (target_state[name] - global_state[name])
    )

torch.save(malicious_state, "attack_aggregation.pt")
```

上传 `attack_aggregation.pt` 后，服务器再除以客户端数量 `10`，放大的恶意差分恰好把聚合结果拉到目标后门模型附近。公开赛后记录确认两项评测均越过阈值，得到：

```text
SUCTF{Faith_can_move_mountains}
```

## 方法总结

这是典型的 federated model replacement。真正的问题不是“怎样把后门训练得很强”，而是服务器把所有客户端更新无条件线性平均，攻击者因此可以按聚合公式反解并放大自己的更新。更稳健的服务端至少应做更新范数裁剪、异常方向检测、鲁棒聚合和独立的后门验证；否则只要攻击者知道客户端数量和初始模型，简单平均就会成为精确的模型替换通道。
