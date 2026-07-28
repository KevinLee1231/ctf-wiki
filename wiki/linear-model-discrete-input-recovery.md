---
type: technique
tags: [ai-ml, linear-model, discrete-input, smt, lattice, input-recovery]
skills: [ctf-ai-ml, ctf-crypto]
raw:
  - ../raw/ai-ml/SUCTF2026-babyAIWP.md
  - ../raw/ai-ml/UMDCTF2023-elgyems-password-wp.md
updated: 2026-07-28
---

# Linear-Model Discrete-Input Recovery

## 适用场景

模型权重已知，网络没有有效非线性或可折叠成仿射映射；公开输出约束了未知输入字节。根据是否存在模运算和小噪声，分别用精确 SMT/整数约束或 CVP/BDD 格恢复离散输入。

## 识别信号

- 多层 `Linear` 或 `Conv -> Flatten -> Linear` 之间没有 ReLU、pooling、softmax 等非线性。
- 权重是精确整数/有理数，或模 `q` 系数与小噪声。
- 直接求逆/伪逆得到非整数、多解或不满足字符范围。
- flag 前后缀、ASCII 范围和输入长度提供强离散约束。

## 最小证据

- 已把完整网络折叠为 `y = Ax + b` 或 `y = Ax + b + e (mod q)`。
- 明确权重精度、模数、噪声界、输入域和已知字符。
- 候选输入回代后精确满足等式，或所有 centered residual 都落在噪声界内。

## 解法骨架

1. 从 checkpoint/文本模型提取权重与 bias，逐层合并仿射映射。
2. 扣除 bias、已知前后缀和固定输入位置的贡献。
3. 无模数且权重可精确表示时，用 Z3 整数/有理数变量加入字符域与格式约束。
4. 模 `q` 且有小噪声时，将字符中心化并构造 CVP/BDD 格。
5. 由 LLL/BKZ + Babai/CVP 得到候选，必要时枚举邻近向量。
6. 用原模型和输出格式做端到端验证。

## 关键变体

| 证据模型 | 首选方法 | 关键约束 |
|---|---|---|
| 精确有理仿射、无噪声 | SMT/整数线性约束 | 字符范围、前后缀、全部输出等式。 |
| 模 `q` 线性、小噪声 | CVP/BDD、Babai | centered mod、噪声界、变量中心化。 |
| 欠定实数系统 | 不直接用伪逆 | 依赖离散域与格式恢复唯一合法解。 |
| checkpoint 权重 | 展开 state_dict | 先确认 shape、布局和卷积窗口。 |

## 常见陷阱

- 看到模型就走 adversarial optimization，忽略其实际上是线性方程组。
- 用 float 近似破坏本可精确表示的有理关系。
- 在精确无噪声题上强行建格，增加不必要的不确定性。
- 不扣除已知字符贡献，导致格维度和目标偏移过大。

## 关联技巧

- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [constraint-solver-and-symbolic-state-recovery.md](constraint-solver-and-symbolic-state-recovery.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [SUCTF2026-babyAIWP](../raw/ai-ml/SUCTF2026-babyAIWP.md)
- [UMDCTF2023-elgyems-password-wp](../raw/ai-ml/UMDCTF2023-elgyems-password-wp.md)
