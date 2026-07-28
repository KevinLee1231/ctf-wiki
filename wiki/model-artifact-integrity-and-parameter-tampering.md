---
type: technique
tags: [ai-ml, model-artifact, integrity, state-dict, parameter-tampering]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/0xGame2025-week4-旧吊带袜天使想吃真蛋糕的Stocking-wp.md
updated: 2026-07-28
---

# Model-Artifact Integrity and Parameter Tampering

## 适用场景

服务允许上传 checkpoint、`state_dict`、LoRA 或其它模型参数，并在固定架构中加载；即使反序列化限制阻止对象级代码执行，攻击者仍可用结构合法但语义恶意的权重直接控制分类、阈值或安全决策。

## 识别信号

- 服务使用 `torch.load(..., weights_only=True)`，随后 `load_state_dict()` 到固定网络。
- 上传内容必须保持全部键名和 tensor shape，但参数值未做签名或可信来源验证。
- 成功条件只依赖一个类别、logit、阈值或排序。
- 最后线性层的权重/偏置足以压过前面所有特征。

## 最小证据

- 从模型类或合法样本确认完整 key、shape、dtype 与 `strict` 行为。
- 证明目标输出可由某一层参数决定，并说明其它层为何不影响结果。
- 恶意制品能被限制模式正常加载，且不依赖 pickle/object RCE。
- 用多个输入验证结果稳定，不是对单一样本的偶然过拟合。

## 解法骨架

1. 获取与服务完全一致的模型类，生成完整合法 `state_dict`。
2. 定位最终决策的 logit、类别索引和服务端阈值。
3. 将目标输出层权重置零或缩放，利用 bias 固定类别排序；必要时在更早层构造恒定激活。
4. 保持所有键、shape、dtype 和设备兼容，保存纯 tensor 制品。
5. 本地用服务同样的加载与预处理流程测试多组输入，再上传触发。

## 关键边界

- `weights_only=True` 限制可反序列化对象，不验证 tensor 参数可信性。
- 这是模型制品完整性问题，不是 pickle 反序列化 RCE。
- 直接改权重固定输出，不等于训练数据投毒或触发器后门；后者需要训练/触发模式证据。
- `strict=True` 只验证 key 与 shape，不验证模型行为。

## 常见陷阱

- 继续寻找 pickle gadget，忽略服务已明确使用安全对象白名单。
- 只上传最后一层，因 `strict=True` 缺少其它 key 而失败。
- 改错类别索引或没有复现服务端 softmax/阈值。
- 用极端值造成 overflow/NaN，导致判定不稳定。

## 关联技巧

- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [model-backdoor-trigger-and-poisoning-analysis.md](model-backdoor-trigger-and-poisoning-analysis.md)
- [linear-model-discrete-input-recovery.md](linear-model-discrete-input-recovery.md)
- [adversarial-ml.md](adversarial-ml.md)

## 原始资料

- [0xGame2025-week4-旧吊带袜天使想吃真蛋糕的Stocking-wp](../raw/ai-ml/0xGame2025-week4-旧吊带袜天使想吃真蛋糕的Stocking-wp.md)
