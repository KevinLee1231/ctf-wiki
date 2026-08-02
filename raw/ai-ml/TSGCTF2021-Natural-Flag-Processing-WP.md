# TSGCTF2021 Natural Flag Processing WP

## 题目简述

题目要求找到一个能被 PyTorch RNN 模型判定为真的 flag。模型由输入维度为字符表大小、隐藏维度为 520 的 `nn.RNNCell` 和一个单输出线性层组成；输入字符经过 one-hot 编码，并在首尾补上 `^`、`$`。

这并不是需要训练替代模型或反复查询黑盒的任务。生成器把一个确定有限自动机直接编码进稀疏权重矩阵：隐藏单元代表状态，输入权重代表触发字符，循环权重代表状态转移，输出层则标记接受状态。因此只要读取模型权重，就能沿接受状态的前驱链逆向恢复唯一字符串。本地仓库快照缺少发布时使用的 `model_final.pth`，所以无法在当前快照中重新加载模型；不过生成源码、官方求解器、明文 flag 和一页自动机 PDF 彼此吻合，足以完整还原机制与求解步骤。

## 解题过程

字符集合定义为：

```python
FLAG_CHARS = string.ascii_letters + string.digits + "{}-"
CHARS = "^$" + FLAG_CHARS
```

输入文本先变为 `^` + flag + `$`，再逐字符送入 RNNCell。普通训练得到的 RNN 权重通常是稠密浮点数，难以直接解释；本题的生成器却人为构造了稀疏矩阵。把循环权重记为 $W_h$、输入权重记为 $W_i$：

- `model.out.weight` 中最大权重所在的隐藏单元是接受状态；
- 对当前状态 `state`，`argmax(W_h[state, :])` 给出它在主路径上的前驱状态；
- `argmax(W_i[state, :])` 给出进入该状态所需的字符；
- 当前行对应的最大循环权重为 0 时，说明已经回到起始端，可以停止回溯。

官方 PDF 只有一页，是一张纵向很长的状态机图。逐页视觉核对可见，黄色节点和红色边构成从 `^` 到 `$` 的主链，周围大量回边与分支把错误输入送往非接受路径。这张图的有用信息正是“唯一主链加干扰转移”，代码截图式保留反而不便阅读，因此将其转写为上述文字关系。

逆向权重的核心代码如下：

```python
state = model.out.weight.data[0, :].argmax()
answer = ""

while True:
    prev_state = Wh[state, :].argmax()
    answer += CHARS[Wi[state, :].argmax()]
    if Wh[state, prev_state] == 0:
        break
    state = prev_state

flag = answer[-2:0:-1]
print(flag)
```

回溯得到的字符顺序是从接受状态向起点，因此最后要反转，并去掉两端的 `$` 与 `^`。恢复结果为：

```text
TSGCTF{mRNA-st4nDs-f0r-mANuaLLy-tun3d-RecurrEn7-N3uRAl-AutoM4toN}
```

## 方法总结

本题的决定性障碍是模型权重解释，归类为 AI/ML 更合适。对规模不大、权重异常稀疏的神经网络，不应默认把它当成不可解释黑盒；先检查权重分布、最大值位置和连接结构，常能发现其中编码的符号系统。本题实质上是“用 RNN 张量保存 DFA”，求解过程也就转化为从输出接受状态沿 $W_h$ 找前驱、沿 $W_i$ 读字符的图遍历。对缺失模型附件的归档还应明确证据边界，不能伪称已在当前快照运行推理。
