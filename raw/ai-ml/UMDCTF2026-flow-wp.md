# flow

## 题目简述

题目要求修改一组 $5\times64$ 的输入，使其经过流模型净化后被分类器以至少 $0.8$ 的概率判为目标类别。扰动受 $L_\infty\leq0.05$ 限制，原始样本还必须达到流模型的对数概率阈值。

服务端的净化过程连续执行三次：

```python
z = flow(x)
x = flow.inverse(0.55 * z)
```

这个过程会压缩潜变量并削弱普通梯度攻击，因此官方解法使用 BPDA（Backward Pass Differentiable Approximation）。

## 解题过程

官方脚本不直接优化输入扰动，而是先计算参考样本的潜变量 $z_0$，把可训练变量设为潜空间增量 $\Delta z$：

$$
x_\text{decoded}=f^{-1}(z_0+\Delta z),
\qquad
x' = x_0+\operatorname{clip}(x_\text{decoded}-x_0,-0.05,0.05).
$$

前向计算必须使用真实净化结果，否则优化目标与服务端不一致；反向计算则把净化近似为恒等映射，让分类损失的梯度直接传回输入。可以用自定义自动求导函数表达：

```python
class BPDA(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x, purified_value):
        return purified_value

    @staticmethod
    def backward(ctx, grad_output):
        return grad_output, None
```

随后用 Adam 优化 $\Delta z$ 上的目标类别交叉熵。每轮都从潜变量重新解码、投影到合法输入区间，并重新检查未净化输入的流模型对数概率：

```python
z0, _ = flow(x0)
dz = torch.zeros_like(z0, requires_grad=True)
x_decoded = flow.inverse(z0 + dz)
x_adv = x0 + torch.clamp(x_decoded - x0, -0.05, 0.05)
x_pure = BPDA.apply(x_adv, purify_no_grad(x_adv, flow))
loss = F.cross_entropy(classifier(x_pure), target)
opt.zero_grad()
loss.backward()
opt.step()
```

官方脚本不会盲目采用最后一次迭代，而是保存同时满足以下条件的最佳候选：

1. 扰动没有超过 $0.05$；
2. 原始对抗样本的流模型分数通过阈值；
3. 三轮真实净化后的目标类别概率不低于 $0.8$。

按服务端格式提交这些 `float32` 样本后得到：

```text
UMDCTF{id_like_to_thank_athalye_carlini_and_wagner_for_their_research}
```

## 方法总结

BPDA 的用途不是篡改前向结果，而是在反向阶段为难以优化或刻意破坏梯度的防御提供近似导数。前向严格复现服务端净化、反向用恒等梯度，再配合扰动投影和流模型可行性检查，即可稳定生成有效样本。
