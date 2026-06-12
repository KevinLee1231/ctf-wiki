# Residue WP

## 题目简述

题目给出 GPT-2 Medium 对一段隐藏 flag 做前向传播后得到的 logits，目标是在已知模型权重的情况下，从 logits 反推出原始输入文本。题目核心不是训练模型，而是利用固定模型前向传播的确定性和近似可逆性做逐 token 搜索。

这类题的关键点是：LLM 的采样过程有随机性，但模型前向传播本身是确定函数。对固定模型和固定上下文而言，同一个 token 序列会产生唯一的 logits 序列。论文中称这类逐 token 反演方法为 SipIt。

## 解题过程

GPT-2 是自回归模型。设已经恢复出前 `t-1` 个 token，则第 `t` 个位置的 logits 只由历史 token 和当前候选 token 决定。因为 GPT-2 词表规模约为 50257，直接枚举当前 token 并比较 logits 即可。

恢复流程如下：

1. 加载 `target_logits.npy`，同时加载 `gpt2-medium` 的 tokenizer 和模型权重。
2. 从第一个位置开始，维护已经恢复出的 token 序列 `decoded_ids`。
3. 对每个位置 `t`，枚举词表内所有候选 token。
4. 将候选 token 接到已知历史后做一次前向传播，得到当前位置输出 logits。
5. 用 MSE 比较候选 logits 与目标 logits，MSE 接近 0 的候选就是当前位置 token。
6. 固定该 token，更新 KV cache，进入下一轮。

伪代码如下：

```python
target = np.load("target_logits.npy")
decoded_ids = []
past_key_values = None

for t in range(seq_len):
    best_token = None
    best_mse = float("inf")

    for batch in batched(range(tokenizer.vocab_size)):
        input_ids = torch.tensor(batch).unsqueeze(1).to(device)
        outputs = model(input_ids, past_key_values=expand_cache(past_key_values, len(batch)))
        logits = outputs.logits[:, -1, :].double()
        mse = ((logits - target[0, t]) ** 2).mean(dim=1)

        if mse.min() < best_mse:
            best_mse = mse.min()
            best_token = batch[mse.argmin()]

    decoded_ids.append(best_token)
    past_key_values = model(torch.tensor([[best_token]], device=device),
                            past_key_values=past_key_values).past_key_values
```

实际实现时最容易踩坑的是效率。若每次候选 token 都重新计算完整历史，复杂度会非常高。应当缓存已经恢复历史的 `past_key_values`，枚举当前 token 时只扩展 batch 维度，让每个候选共享同一份历史 KV cache。官方题解中提到，使用 KV cache 后普通桌面 GPU 也可以在可接受时间内完成恢复。

## 方法总结

本题不是训练模型或做梯度优化，而是利用“固定 Transformer 前向传播可判别”的性质做逐 token 原像搜索。核心判定标准是 logits 的数值一致性：错误 token 的 logits 与目标有明显差异，正确 token 的 MSE 接近 0。实现上要注意 dtype、模型版本和 tokenizer 必须与出题环境一致，否则浮点差异会导致判定失败。
