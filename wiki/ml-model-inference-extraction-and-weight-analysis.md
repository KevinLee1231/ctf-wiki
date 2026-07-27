---
type: family
tags: [ai-ml, model-extraction, inference, weight-analysis, family]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/model-attacks.md
  - ../raw/ai-ml/LilacCTF2026-residue-wp.md
  - ../raw/ai-ml/SUCTF2026-谁是小偷WP.md
  - ../raw/ai-ml/SUCTF2026-我不是神偷WP.md
  - ../raw/ai-ml/SUCTF2026-babyAIWP.md
  - ../raw/ai-ml/SUCTF2026-easyLLMWP.md
  - ../raw/ai-ml/SUCTF2026-theifWP.md
updated: 2026-07-27
---

# ML Model Inference, Extraction and Weight Analysis

## 作用边界

本页是 AI/ML 模型攻击 family。模型本身是主要证据：需要通过推理接口、输出置信度、任务语义、模型权重、embedding、encoder、LoRA adapter 或训练/推理参数恢复 flag。若核心是 prompt injection，转到 [llm-attacks.md](llm-attacks.md)；若核心是扰动样本欺骗分类器，转到 [adversarial-ml.md](adversarial-ml.md)。

## 识别信号

- 附件包含模型文件、权重、配置、推理服务、输入输出样本、embedding、encoder 或 adapter。
- 成功条件取决于模型输出、隐藏类别、相似度、置信度、梯度、参数差异或训练数据痕迹。
- 服务允许反复查询，且反馈可以被统计、拟合、搜索或反演。
- 题目表面像业务逻辑，但关键判断由模型推理结果或模型内部状态决定。

## 最小证据

- 固定输入格式、预处理、输出字段和评分/判定阈值。
- 记录至少一组可复现输入输出，并说明反馈是离散标签、概率、embedding、loss、文本还是 side channel。
- 若有模型文件，先确认框架、层结构、权重维度、checkpoint/config 关系和是否存在 adapter/patch。
- 能说明当前路径是 query extraction、model inversion、weight diff、encoder collision、membership inference，还是普通 adversarial 样本。

## 分流流程

1. 先做 I/O 契约：输入类型、shape、归一化、tokenizer、输出字段和限制次数。
2. 如果能查询服务，构造少量代表性查询样本，观察标签、概率、长度、错误和边界反馈。
3. 如果有权重或 checkpoint，检查 config、state_dict、异常层、adapter 合并和权重差分。
4. 根据反馈选择路线：抽取近似模型、反演输入、搜索碰撞、恢复隐藏类别、推断训练样本或合并/抵消权重扰动。
5. 输出可复验样本或脚本，证明模型产生目标分类、目标文本、目标 embedding 或恢复出的 flag。

## 模型路线分流

| 模型路线 | 首轮判断 |
|---|---|
| Query-based model extraction | 查询预算、输入覆盖、输出置信度和近似模型评估是核心；先建立可重复采样策略。 |
| Model inversion | 输出概率、embedding 或 loss 可提供梯度/优化目标；先复现预处理和约束范围。 |
| Gradient leakage | 单样本参数梯度、checkpoint 差分或 bias/weight gradient 可直接约束前层激活时，转 [gradient-leakage-input-reconstruction.md](gradient-leakage-input-reconstruction.md)。 |
| Weight analysis / perturbation | 对比 checkpoint、adapter 或权重补丁，找异常矩阵、偏置、符号翻转和可逆扰动。 |
| Encoder collision | 如果成功条件是相同 embedding/latent，优先分析归一化、哈希/量化和距离阈值。 |
| Membership inference | 需要区分训练样本和非训练样本反馈差异，避免把随机高置信误判为成员证据。 |
| Task-semantic inference | 部分题没有复杂 ML 攻击，关键是把输入输出约束形式化成搜索、分类或规则恢复问题。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 参数梯度由单样本生成，需要反演输入/激活 | [gradient-leakage-input-reconstruction.md](gradient-leakage-input-reconstruction.md) |
| 无激活线性层输出足以解出权重/偏置 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |
| 已知 Transformer logits，需要逐 token 恢复原像 | [transformer-logit-inversion.md](transformer-logit-inversion.md) |

## 常见陷阱

- 没有复现预处理就直接优化输入，导致本地结果和服务不一致。
- 把 LLM prompt 绕过放到模型抽取页，造成路线混乱。
- 忽略查询预算和反馈粒度，选择了不可能收敛的攻击路线。
- 只看模型结构，不检查 checkpoint/config/tokenizer 是否匹配。
- 对中文题名或业务叙事过拟合，没有把任务抽象成可测试的输入输出约束。

## 关联技巧

- [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md)
- [linear-model-input-lattice-recovery.md](linear-model-input-lattice-recovery.md)
- [transformer-logit-inversion.md](transformer-logit-inversion.md)
- [gradient-leakage-input-reconstruction.md](gradient-leakage-input-reconstruction.md)
- [adversarial-ml.md](adversarial-ml.md)
- [llm-attacks.md](llm-attacks.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [LilacCTF2026-residue-wp](../raw/ai-ml/LilacCTF2026-residue-wp.md) | 已知 GPT-2 Medium 权重和目标 logits，可逐位置枚举词表并用 MSE 匹配 logits；KV cache 是可接受复杂度的关键。 | [transformer-logit-inversion.md](transformer-logit-inversion.md) |
| [SUCTF2026-谁是小偷WP](../raw/ai-ml/SUCTF2026-谁是小偷WP.md) | `Conv2d -> Flatten -> Linear` 且无激活函数，`/predict` 返回完整向量；先用 basis query 恢复整体仿射映射，再拆参数。 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |
| [SUCTF2026-我不是神偷WP](../raw/ai-ml/SUCTF2026-我不是神偷WP.md) | 附件结构与线上形状冲突，`/flag` 报错暴露真实 state_dict；先恢复共享线性层，再把两层卷积分解为等效核。 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |
| [SUCTF2026-babyAIWP](../raw/ai-ml/SUCTF2026-babyAIWP.md) | `Conv1d -> Linear` 无激活且权重藏在 `model.pth`；展开成带小噪声的模线性方程后用 LLL/Babai 恢复 flag 字节。 | [linear-model-input-lattice-recovery.md](linear-model-input-lattice-recovery.md) |
| [SUCTF2026-easyLLMWP](../raw/ai-ml/SUCTF2026-easyLLMWP.md) | LLM 输出被直接派生为 AES key，且模型、prompt、temperature 和输出格式已知；先采样候选输出并用密文/PKCS#7 oracle 碰撞验证，不要当普通 prompt injection 处理。 | [llm-attacks.md](llm-attacks.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SUCTF2026-theifWP](../raw/ai-ml/SUCTF2026-theifWP.md) | 模型上传接口只比较部分参数，四维卷积权重校验缺口可配合 `/predict` 输出恢复被检查线性层。 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |
| [0xGame2025-week4-旧吊带袜天使想吃真蛋糕的Stocking-wp](../raw/ai-ml/0xGame2025-week4-旧吊带袜天使想吃真蛋糕的Stocking-wp.md) | 本题考查模型参数完整性，而不是不安全反序列化。`weights_only=True` 限制了可反序列化对象，却不会判断张量参数是否可信；只要攻击者能上传结构合法的 `state_dict`，就能通过控制最后一层权重和偏置决定所有分类结果。生产环境需要对模型制品做签名、来源验证和行为测试，不能把“能成功加载”当作“模型可信”。 | 本页对应路线 |
| [MoeCTF2023-A-very-happy-MLP-wp](../raw/ai-ml/MoeCTF2023-A-very-happy-MLP-wp.md) | 本题的关键是区分“求一个输入原像”和“训练模型”。线性层降维后没有唯一逆，但伪逆能给出一个可验证的原像；单调激活可以在值域内逐点求逆。逆推神经网络时应从输出层向输入层逐层进行，并在每一步检查激活函数值域，最后用原模型重新前向验证结果。 | 本页对应路线 |
| [MoeCTF2023-Visual-Hacker-wp](../raw/ai-ml/MoeCTF2023-Visual-Hacker-wp.md) | 本题把分类式角度回归、三维方向向量和射线—平面求交串成一条完整的数据恢复链。容易出错的地方有三个：确认模型输出头的顺序、在推理前切换到 `eval()`、统一摄像机与书写者的坐标系。最终轨迹包含噪声时，应依据整组几何形状识别字符，而不是期待每帧都落在理想笔画上。 | 本页对应路线 |
| [UMDCTF2023-elgyems-password-wp](../raw/ai-ml/UMDCTF2023-elgyems-password-wp.md) | 没有激活函数的深层网络并不增加表达能力，可以精确折叠为一个仿射变换；伪逆只给出某个实数解；flag 格式和可打印 ASCII 是选出离散原像的必要约束。 | 本页对应路线 |
| [UMDCTF2024-llm-compression-wp](../raw/ai-ml/UMDCTF2024-llm-compression-wp.md) | 把语言模型视为自适应概率源，使用同一模型权重驱动算术解码器逐字节重建原文；附件同时包含 Transformer 参数和短小高熵二进制流，题面又明确提到“language modeling is compression”时，应寻找模型概率与熵编码的组合，而不是把 `.npz` 当普通隐藏压缩包。 | 本页对应路线 |
| [UMDCTF2025-italian-ai-brainrot-wp](../raw/ai-ml/UMDCTF2025-italian-ai-brainrot-wp.md) | DDIM inversion 恢复初始高斯 latent，逆 Box–Muller 型采样得到 MT19937 部分输出，再解 $\mathbb F_2$ 线性系统伪造 PyTorch RNG 状态。 | 本页对应路线 |
| [WMCTF2020-performance-artist-wp](../raw/ai-ml/WMCTF2020-performance-artist-wp.md) | 图像分类题要先判断样本是否来自公开训练集、是否加噪、是否需要训练模型。本题因为直接使用训练集，核心不是泛化分类，而是样本匹配和载体修复。这里的可复用点是把 `0-9` 和 `a-f` 统一成 16 类，把图块分类结果按十六进制半字节还原文件。若图片中疑似还藏有压缩包，需要同时检查 PNG 高度、文件尾、ZIP magic 和伪加密位，避免把分类结果和容器修复割裂开。 | 本页对应路线 |
| [UMDCTF2023-torchics-revenge-wp](../raw/reverse/UMDCTF2023-torchics-revenge-wp.md) | 从框架 API 调用和维度常量恢复模型拓扑，再用序列化对象大小验证权重与偏置顺序；巨大的二进制尺寸与 `serde_pickle` 调用共同提示模型数据被直接嵌入。 | 本页对应路线 |

## 原始资料

- [model-attacks.md](../raw/ai-ml/model-attacks.md)
- [LilacCTF2026-residue-wp](../raw/ai-ml/LilacCTF2026-residue-wp.md)
- [SUCTF2026-谁是小偷WP](../raw/ai-ml/SUCTF2026-谁是小偷WP.md)
- [SUCTF2026-我不是神偷WP](../raw/ai-ml/SUCTF2026-我不是神偷WP.md)
- [SUCTF2026-babyAIWP](../raw/ai-ml/SUCTF2026-babyAIWP.md)
- [SUCTF2026-easyLLMWP](../raw/ai-ml/SUCTF2026-easyLLMWP.md)
- [SUCTF2026-theifWP](../raw/ai-ml/SUCTF2026-theifWP.md)
