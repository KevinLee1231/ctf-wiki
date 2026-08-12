# SABLE

## 题目简述

题目给出一个基于图注意力的垃圾账号判定器。目标节点 `user_target` 的原始风险很高，选手可以提交至多六个新好友节点；每个节点必须带有八个归一化特征，并且每条新边只能把该节点连到目标。校验器先执行公开的合理性、取值范围和去重检查，再把扩展后的目标邻域送入公开的 PyTorch 模型；当风险概率小于 $0.05$ 时接受提交。

模型不是按邻居逐个投票，而是将目标特征投影为 query、邻居特征投影为 key/value，再计算单头 scaled dot-product attention：

$$
\operatorname{Attn}(q,K,V)=\operatorname{softmax}\left(\frac{qK^T}{\sqrt{4}}\right)V.
$$

因此，能够伪造的好友节点既会改变 value 的风险侧特征，也会通过 key 抢走原有高风险邻居的注意力权重。决定性障碍是公开模型权重下的可微优化和约束满足，而非 JSON 解析或服务接口，故归入 `ai-ml`。

## 解题过程

### 还原提交约束与模型目标

`graph_utils.py` 固定了字段顺序：`post_rate_norm`、`profile_age_norm`、`report_rate`、`external_link_rate`、`profile_realness`、`shared_audience_overlap`、`interaction_strength`、`trust_score`。前两个字段分别受 $[0,0.55]$、$[0,0.45]$ 约束，其余字段受 $[0,1]$ 约束。提交还必须满足：

- 最多六个 `friend_*` 节点，每个节点恰好一条到 `user_target` 的边；
- 节点全特征 L1 距离至少为 $0.055$，六个活跃特征上的 L1 距离至少为 $0.020$；
- 高 `shared_audience_overlap` 或 `interaction_strength` 必须配套足够的 `report_rate`、`external_link_rate`；过于完美且强关联的档案也会被拒绝。

公开模型的核心足够小，可以直接对待提交的六个 $8$ 维向量求梯度：

```python
q = q_proj(target_x).view(1, 1, 1, 4)
k = k_proj(neighbor_x).view(1, 1, neighbor_x.shape[0], 4)
v = v_proj(neighbor_x).view(1, 1, neighbor_x.shape[0], 4)
attended = F.scaled_dot_product_attention(q, k, v, dropout_p=0.0)
logit = classifier(torch.cat([tanh(target_proj(target_x)), attended.view(4)]))
risk = sigmoid(logit)
```

所以目标是最小化 `logit`，并非直接构造畸形 JSON。直接把所有风险相关特征压到零会触发合理性检查，也容易生成近重复好友。

### 在可行域中进行梯度优化

为保证每一次迭代均处于字段边界内，将无约束变量 $z$ 映射为

$$
x_j=l_j+\sigma(z_j)(u_j-l_j).
$$

对六个节点运行 Adam，损失由模型 logit、镜像化的软约束惩罚和很小的真实度/信任度奖励组成：

```python
friends = bounded_features(raw)
neighbor_x = torch.cat([base_neighbor_x, friends], dim=0)
logit = model(target_x, neighbor_x)
idx = {name: FEATURE_NAMES.index(name) for name in FEATURE_NAMES}
loss = (
    logit
    + 600.0 * constraint_penalty(friends)
    - 0.012 * (
        friends[:, idx["profile_realness"]].mean()
        + friends[:, idx["trust_score"]].mean()
    )
)
loss.backward()
opt.step()
```

`constraint_penalty` 对校验器的联动下限、过度完美规则和两种 L1 多样性规则分别加入 ReLU 平方惩罚。优化后不能只信连续损失；还要把候选投影到稳定的离散可行脊线上，重新经过原校验器。

该脊线令 `shared_audience_overlap` 递增为 $0.88,0.90,\ldots,0.98$，`interaction_strength` 递减为 $0.84,0.82,\ldots,0.74$，同时按公开公式抬高两种风险遥测下限：

$$
\begin{aligned}
\text{report\_rate}&\ge 0.36\max(0,\text{overlap}-0.70),\\
\text{external\_link\_rate}&\ge 0.36\max(0,\text{interaction}-0.70),\\
\text{report\_rate}+\text{external\_link\_rate}&\ge 0.24\max(0,\text{overlap}+\text{interaction}-1.30).
\end{aligned}
$$

`profile_realness` 与 `trust_score` 保持接近 $1$，而 `post_rate_norm`、`profile_age_norm` 使用梯度无关的不同值来稳定通过 L1 多样性检查。这样伪造节点与目标 query 的 key 兼容度高，又使 value 侧表示偏向低风险，从而稀释原有危险邻居。

### 生成并验证 payload

官方求解器会从公开的 `model.pt` 和 `public_graph.json` 重新生成六节点 payload，而不是依赖已保存的答案。随后它调用同一份公开 `server.py`：

```bash
python solve/solve.py
python dist/server.py solve/payload.json --debug
```

校验器要求基线风险至少为 $0.75$，再检查优化后的 `risk < 0.05`。`--debug` 同时列出注意力权重；成功 payload 中新增 `friend_00` 到 `friend_05` 占据主要注意力质量，证明降风险来自注意力稀释而非绕过验证。远端返回的 flag 为 `grey{w40w_Y0u_h4Z_a_L0t_oF_Fr3n5_inDeEd_:0}`。

## 方法总结

- 核心技巧：把可添加节点视为 GNN 图注意力的 Sybil 注入变量，联合优化 key 的注意力竞争和 value 的分类方向。
- 识别信号：公开权重、可控邻居特征、有限节点预算以及“合理性”校验同时出现时，应检查是否能在可行域内重分配 attention mass。
- 复用要点：硬校验最好在优化时镜像成软惩罚，最终必须回到原校验器验证。用于拉开样本距离、却不进入模型的字段可作为稳定通过多样性限制的自由度。
