---
type: technique
tags: [ai-ml, model-extraction, linear-model, parameter-recovery, pytorch]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/SU_谁是小偷WP.md
  - ../raw/ai-ml/SU_我不是神偷WP.md
  - ../raw/ai-ml/SU_theifWP.md
updated: 2026-07-06
---

# Linear Model Parameter Recovery

## 适用场景

目标模型本身就是主要证据，且推理行为足够线性或近似线性。典型形态是 `Conv2d -> Flatten -> Linear`、没有激活函数、`/predict` 返回完整向量，或者 `/flag` 通过 state_dict 逐参数比较用户上传模型。

本页不处理对抗样本扰动，也不处理 LLM prompt 绕过。若成功条件是改变分类边界但不需要恢复参数，转 [adversarial-ml.md](adversarial-ml.md)；若是 prompt/tool-use 语义，转 [llm-attacks.md](llm-attacks.md)。

## 识别信号

- `/predict` 返回完整 logits / embedding / 256 维向量，而不是只有标签。
- 模型没有 ReLU、pooling 或非线性层，整体可视为仿射映射。
- 附件模型结构和线上报错不一致，`/flag` 或加载错误泄露真实 shape。
- 上传接口用 `torch.load(..., weights_only=True)` 或 state_dict 参数比较，比较逻辑遗漏某类权重、维度或卷积核。
- 多个服务版本共享部分层，历史服务可作为恢复当前服务参数的锚点。

## 最小证据

- 已确认真实输入 shape、输出维度、预处理和 `/predict` 反馈粒度。
- 至少能构造一组基向量或线性无关输入，并记录对应输出。
- 已判断模型中哪些参数会被 `/flag` 严格比较，哪些参数没有被比较或只被弱约束。
- 能用本地 PyTorch forward 复现至少一组远程输出，避免把附件里的烟雾结构当真。

## 解法骨架

1. 先以线上行为为准：用合法/非法输入、`/flag` 报错和 state_dict 加载错误确定真实 shape。
2. 把模型写成仿射形式。无激活网络可整理为 `y = T x + b`，卷积层和线性层组合后仍可用查询恢复整体映射。
3. 通过 basis query 或线性无关样本恢复 `T` 和 `b`；输出维度较大时记录查询矩阵是否满秩。
4. 若需要拆回卷积和线性层，先用零空间、SVD、伪逆或等效卷积核分解恢复一组满足 forward 的参数。
5. 若上传校验只检查部分参数，优先恢复被检查层；未检查层可以沿用附件、构造等效值或满足 shape 即可。
6. 最后上传 state_dict 前，本地 forward check 远程样本；上传失败时先看误差阈值、dtype、序列化格式和参数名。

## 关键变体

| 变体 | 判断方式 | 处理 |
|---|---|---|
| 单层卷积加线性层 | `/predict` 完整输出，模型无激活 | basis query 恢复整体仿射映射，再用零空间/SVD 分解卷积和线性层。 |
| 多服务共享层 | 旧服务和当前服务结构不同，但部分输出行为或线性层一致 | 先恢复共享层，再把当前卷积组合视为等效核并做因式分解。 |
| 上传校验漏权重 | state_dict 比较覆盖线性层或 bias，但遗漏四维卷积权重 | 只恢复被检查层，未检查层用能通过 forward 或 shape 的参数填充。 |
| 附件结构是烟雾 | 附件 `app.py` 与线上报错 shape 冲突 | 不按附件推导，先由线上报错、合法输入和返回维度反推真实结构。 |

## 常见陷阱

- 先相信附件里的模型结构，没有用线上 `/predict` 和 `/flag` 报错确认真实 shape。
- 把 `weights_only=True` 当成反序列化利用点，忽略题目真正考的是参数比较逻辑。
- 只恢复整体矩阵，却没拆回服务端期望的 state_dict 参数名和维度。
- 线性无关样本数量不够，导致伪逆/分解结果看似可用但上传误差超阈值。
- 本地 forward 使用的 dtype、归一化、flatten 顺序和远程不一致。

## 关联技巧

- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [adversarial-ml.md](adversarial-ml.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [SU_谁是小偷WP](../raw/ai-ml/SU_谁是小偷WP.md)
- [SU_我不是神偷WP](../raw/ai-ml/SU_我不是神偷WP.md)
- [SU_theifWP](../raw/ai-ml/SU_theifWP.md)
