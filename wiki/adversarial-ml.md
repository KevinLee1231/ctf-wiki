---
type: family
tags: [ai-ml, family, adversarial-examples, evasion, poisoning, backdoor]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/adversarial-ml.md
updated: 2026-07-28
---

# Adversarial ML

## 作用边界

本页是 adversarial examples、evasion、patch、poisoning 和 backdoor 检测 family。它负责从模型、输入约束、梯度可用性和评分反馈判断应走 FGSM/PGD/C&W、adversarial patch、black-box 查询、数据投毒还是后门触发器分析。

## 识别信号

- 成功条件是让分类器误判、通过模型认证、改变置信度、触发后门或规避检测。
- 题目给出模型文件、推理接口、训练数据、label、loss、梯度、输入范围或查询次数限制。
- 输入是图片、音频、特征向量或嵌入，且有像素/范数/格式/肉眼可读性约束。

## 最小证据

- 模型框架、输入 shape、归一化、label space、目标/非目标攻击要求。
- 是否 white-box：能否拿梯度、loss、权重；black-box 时反馈是 label、score 还是二值。
- 扰动约束：Linf/L2/L1、像素范围、mask、patch 位置、查询预算和提交格式。
- 成功样本能被本地或远端稳定复验。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| white-box 分类器 | loss、梯度和预处理是否一致 | FGSM/PGD/C&W |
| 只给 label/score API | 查询预算、score 粒度和随机性 | black-box search / NES / boundary |
| patch 可见区域 | patch 位置、大小和变换增强 | adversarial patch |
| MNIST/图像认证 | 输入 clamp、reshape 和归一化 | foolbox/hand-rolled gradient |
| poisoning/backdoor | 训练数据可控或触发器存在 | 触发器搜索/clean-label 判断 |
| backdoor detection | 模型在带 trigger 输入上稳定偏向目标类，或 activation cluster 出现异常簇 | trigger search / neuron activation / clean-vs-trigger 对比 |
| 有梯度但存在量化、随机变换、净化器或多约束 | 真实前向、替代梯度、投影口径和落盘重载是否一致 | [constraint-aware-adversarial-optimization-and-validation.md](constraint-aware-adversarial-optimization-and-validation.md) |
| 需要模型参数/权重 | 攻击面不是扰动而是模型恢复 | [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 有梯度但量化、随机变换或多约束导致“优化成功、提交失败” | [constraint-aware-adversarial-optimization-and-validation.md](constraint-aware-adversarial-optimization-and-validation.md) |
| 只有 label、score 或二值反馈，需要在查询预算内搜索 | [black-box-query-feedback-evasion.md](black-box-query-feedback-evasion.md) |
| clean 输入正常，局部 trigger、污染样本或异常 activation 稳定导向目标类 | [model-backdoor-trigger-and-poisoning-analysis.md](model-backdoor-trigger-and-poisoning-analysis.md) |
| 攻击者可上传结构合法的 checkpoint/state_dict 并直接控制参数 | [model-artifact-integrity-and-parameter-tampering.md](model-artifact-integrity-and-parameter-tampering.md) |

## 来自 raw 的模式索引

| Raw 模式 | 可复用联系 |
|---|---|
| FGSM / PGD / C&W | white-box 图像分类器优先确认 loss、target/untargeted、epsilon、clamp 和归一化；C&W 更适合要求低可见扰动或 logits margin 的题。 |
| Adversarial patch | patch 位置、mask、缩放/旋转增强和跨样本泛化比单张图片扰动更重要；提交前要按远端预处理复验。 |
| Evasion on classifiers | Malware/text/image 分类器 evasion 的共同点是先找可改特征和保持语义/格式合法性，不直接套图像 FGSM。 |
| Data poisoning / backdoor detection | 训练集或触发器可控时，先验证 clean accuracy 与 trigger success 的差异，再用 activation clustering 或 trigger search 判断后门。 |
| foolbox L1BasicIterativeAttack on Keras MNIST-Auth | 老 Keras/TensorFlow 环境可直接用 foolbox 2.x 生成符合 challenge 序列化格式的样本；关键是版本和输入编码一致。 |
| Hand-rolled Keras FGSM via `K.gradients` | 当现成库版本不匹配时，用 symbolic gradient 手写目标 loss，比迁移新框架更稳。 |
| [WMCTF2020-music-game-wp](../raw/ai-ml/WMCTF2020-music-game-wp.md) | 语音识别类杂项题先判断难点在模型鲁棒性还是上传流程。本题识别模型本身较弱，优先用标准 TTS 生成更接近训练分布的音频；若前端限制录音或上传，可抓包后直接提交音频文件。不要过早转向复杂对抗样本，除非普通 TTS 已无法满足目标。 |
| [WMCTF2020-music-game2-wp](../raw/ai-ml/WMCTF2020-music-game2-wp.md) | 语音模型对抗题的识别信号是：给出模型、样例音频和目标分类，且普通录音/TTS 已不足以稳定通过。可复用流程是先把音频转为模型实际使用的特征表示，例如 MFCC；在特征空间按目标类别反向传播；再把特征近似还原成 wav 并重新送入模型验证。随机加噪可作为低成本碰撞尝试，但稳定解应保留 loss、目标类别、梯度更新和上传验证流程。 |
| [WMCTF2024-easy-num-wp](../raw/ai-ml/WMCTF2024-easy-num-wp.md) | 阈值条件完全暴露，适合直接做白盒对抗样本；均值约束要求扰动后整体偏大，可通过调大 `epsilon` 或多次随机初始点解决。 |

## 合并与拆分结论

- 保留为 family：adversarial ML 覆盖多种攻击目标和反馈模型，适合二级分流。
- 不合并进 LLM 页：LLM prompt/tool 攻击的证据和验证方式不同。
- FGSM/PGD/C&W 的共同约束验证保留在现有 technique；黑盒反馈和后门/投毒已拆为独立 technique，因为它们的最小证据与失败转向不同。

## 常见误判

- 本地预处理和远端不一致，导致样本本地成功远端失败。
- 梯度攻击后未 clamp 或量化，提交格式破坏扰动。
- Black-box 题没有估查询预算，过早使用高成本搜索。
- 只追求误分类，没有满足目标 label 或业务阈值。

## 关联页面

- [llm-attacks.md](llm-attacks.md)
- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [constraint-aware-adversarial-optimization-and-validation.md](constraint-aware-adversarial-optimization-and-validation.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [adversarial-ml.md](../raw/ai-ml/adversarial-ml.md)
