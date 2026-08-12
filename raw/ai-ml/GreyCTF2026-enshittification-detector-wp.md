# Enshittification Detector

## 题目简述

题目给出一个四分类工单模型的候选样本、128 个影子模型及各影子模型的训练成员清单。远端会为当前队伍生成一个审计包，其中包含目标模型 `target.pt`、64 个待判断候选的编号和一次性 `nonce`。需要判断每个候选是否参与了目标模型训练；题目保证成员与非成员各占一半。

核心不是重新训练分类器，而是对每个候选执行成员推断：比较目标模型在该样本上的表现更像“训练时见过它”的影子模型，还是更像“训练时没见过它”的影子模型。

## 解题过程

先从 `candidates.npz` 读取特征、真实标签和候选编号，再根据 `shadow_train_ids.json` 把每个候选在 128 个影子模型中的状态分成 `IN` 与 `OUT` 两组。模型输出不能直接用统一置信度阈值判断，因为有些样本天然容易，没参与训练也会得到很高置信度；必须针对每个候选分别校准。

官方解法选取负交叉熵作为单值统计量：

$$
s(f,x)=-\operatorname{CE}(f(x),y)
$$

对候选 $x$，分别收集已知成员影子模型和非成员影子模型的统计量：

$$
S_{\mathrm{in}}(x)=\{s(f_i,x)\mid x\in D_i\},\qquad
S_{\mathrm{out}}(x)=\{s(f_i,x)\mid x\notin D_i\}
$$

为两组样本各拟合一维高斯分布。将目标模型对同一候选的统计量记为 $s_t$，计算对数似然比：

$$
\Lambda(x)=\log p(s_t\mid S_{\mathrm{in}}(x))-\log p(s_t\mid S_{\mathrm{out}}(x))
$$

实现时给标准差设置 $10^{-3}$ 的下限，避免方差过小导致数值异常。随后按 $\Lambda(x)$ 从大到小排列审计包中的 64 个候选。由于题目明确是平衡集合，直接把前 32 个标为 `IN`，其余标为 `OUT`，比机械使用 $\Lambda(x)>0$ 更稳妥。

核心逻辑可概括为：

```python
for candidate_id in audit_ids:
    idx = candidate_index[candidate_id]
    member = shadow_scores[shadow_membership[:, idx], idx]
    nonmember = shadow_scores[~shadow_membership[:, idx], idx]
    score = gaussian_logpdf(target_scores[idx], member)
    score -= gaussian_logpdf(target_scores[idx], nonmember)
    ranked.append((score, candidate_id))

ranked.sort(reverse=True)
in_ids = {candidate_id for _, candidate_id in ranked[:len(ranked) // 2]}
```

最后按接口要求提交队名、审计包中的 `nonce` 和完整预测表：

```json
{
  "team": "TEAM_NAME",
  "nonce": "NONCE_FROM_MANIFEST",
  "predictions": {
    "candidate_000": "IN",
    "candidate_001": "OUT"
  }
}
```

服务端验证全部标签后返回：

```text
grey{https://www.youtube.com/watch?v=T4Upf_B9RLQ}
```

## 方法总结

这题复现了 LiRA 风格的成员推断。决定性步骤不是寻找一个适用于所有样本的损失阈值，而是利用影子模型为每个候选建立成员与非成员两种条件分布，再做似然比检验。题目额外给出的 1:1 先验应当直接用于最终排序：它既消除了阈值选择问题，也把连续攻击分数稳定地转成了完整二元标签集。
