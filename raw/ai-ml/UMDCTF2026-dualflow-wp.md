# dualflow

## 题目简述

题目给出两个基于 RealNVP 的流模型和一组形状为 $5\times64$ 的参考样本。提交的 `float32` 样本必须同时满足：

- 与参考样本的 $L_\infty$ 距离不超过 $0.08$；
- 在两个模型之间取得足够大的对数似然优势；
- 仍通过目标模型的真实性阈值；
- 目标模型的 log-determinant 落在指定区间。

这不是单纯把分类损失推高的普通对抗样本题。真实性和 Jacobian 约束会抵消直接沿主目标梯度更新的效果，官方解法因此使用梯度正交化。

## 解题过程

记两个流模型给出的对数概率为 $\log q_0(x)$ 和 $\log q_1(x)$，主目标为

$$
M(x)=\log q_1(x)-\log q_0(x).
$$

此外还要控制 $\log q_1(x)$ 和模型 1 的 log-determinant。每轮分别求出三个梯度：

$$
g_m=\nabla_x M,\qquad
g_r=\nabla_x\log q_1,\qquad
g_j=\nabla_x\operatorname{logdet}_1.
$$

直接沿 $g_m$ 更新很容易越过后两个约束。官方脚本把整个 $5\times64$ 窗口展平为一个向量，再用 Gram-Schmidt 将 $g_m$ 在 $g_r$、$g_j$ 张成的方向上逐次消去：

```python
direction = g_margin
for guard in (g_realism, g_logdet):
    guard_hat = guard / (guard.norm() + 1e-12)
    direction -= (direction @ guard_hat) * guard_hat
```

这样得到的方向一阶近似下不会明显改变真实性和 log-determinant，却仍保留提高 margin 的分量。随后做投影梯度更新：

```python
delta += 0.002 * direction / (direction.norm() + 1e-12)
delta.clamp_(-0.08, 0.08)
x = x_ref + delta.reshape_as(x_ref)
```

优化过程中不能只保存 margin 最大的样本，而应在每轮重新计算全部服务端条件，只保留同时可行的候选。若某个样本的真实性或 log-determinant 已接近边界，还可以减小步长，避免浮点误差导致本地通过、远端失败。

将最终张量按服务端要求转换为 `float32` 并提交，得到：

```text
UMDCTF{a_little_gram_schmidt_never_hurt_anybody}
```

## 方法总结

本题的关键是把多个约束视为局部几何问题：主目标梯度负责前进，约束梯度描述暂时不能穿过的方向。对主梯度做 Gram-Schmidt 投影后，再投影回 $L_\infty$ 邻域，能够在不破坏流模型约束的前提下持续扩大两个模型的似然差。
