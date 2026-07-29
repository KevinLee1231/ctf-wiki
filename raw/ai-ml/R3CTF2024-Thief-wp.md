# Thief

## 题目简述

服务加载一个用于 CIFAR-100 的 ResNet-18，从模型实际训练成员中随机抽取 250 张图，再从非成员中抽取 250 张。对每张图只公开 softmax 概率最高的 10 项，要求判断它是否在训练集中。500 题正确率必须严格大于：

$$
0.75+0.1=0.85.
$$

这是一道黑盒成员推断题。模型对见过的训练样本明显更自信，只需要利用最大 softmax 概率，无需恢复图像或模型权重。

## 解题过程

附件 `run.py` 的关键逻辑为：

```python
trained = random.sample(train_indices, 250)
untrained = random.sample(
    list(set(range(10000)) - set(train_indices)), 250
)

output = rec_model(image)
prob = F.softmax(output, dim=1)
top_10, _ = torch.topk(prob, 10)
print(top_10.tolist())
```

服务已经按降序输出 top-10，因此第一个浮点数就是：

$$
p_{\max}=\max_y P(y\mid x).
$$

训练成员受到过拟合影响，通常出现接近 1 的最大置信度；非成员的分布更平缓。公开实例中阈值 0.9 能稳定超过要求：

$$
\hat m(x)=
\begin{cases}
1,&p_{\max}\ge0.9,\\
0,&p_{\max}<0.9.
\end{cases}
$$

交互脚本只需准确提取嵌套列表中的首项：

```python
import re
from pwn import remote

io = remote(HOST, PORT)

for _ in range(500):
    io.recvuntil(b"top_10_pred : ")
    line = io.recvline().decode()
    first = float(re.findall(
        r"[-+]?(?:\d+\.\d+|\d+)(?:e[-+]?\d+)?",
        line,
        flags=re.I,
    )[0])

    answer = b"1" if first >= 0.9 else b"0"
    io.sendlineafter(
        b"Is this picture in the training set?",
        answer,
    )

print(io.recvall().decode())
```

500 个样本中成员与非成员各占一半，所以也可以先观察前几十个 `p_max` 是否形成明显双峰，再把阈值放在两簇之间。不要按样本索引猜成员关系：远端的 `train_indices.csv` 不公开，且题目每次会重新抽样并打乱。

公开复现记录中的一个服务实例返回：

```text
R3CTF{caIN_liKe_A1_4nd_rEC_8772b609d39f}
```

服务端实际从 `GZCTF_FLAG` 环境变量读取 flag；如果平台按实例或队伍动态生成 flag，最终字符串应以当前实例为准，攻击方法不变。

## 方法总结

最大置信度阈值是最基础的 membership inference attack：训练样本的经验损失通常更低，softmax 输出更尖锐。题目已经替选手完成模型查询并公开 top-10，因此第一项就是足够强的统计量。实际评测条件是正确率严格大于 0.85，恰好 425/500 不会通过；脚本和阈值应留出余量，并用科学计数法兼容的解析方式避免把输出格式错误当成模型误判。
