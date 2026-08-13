# GreyCTF2023 PLCG

## 题目简述

服务从公开集合 `sample = [3, 80, r1, r2]` 中反复取值，并迭代

$g\leftarrow(ag+b)\bmod256$

生成每个“随机”字节。六个输出字节经过 PKCS#7 填充后作为 AES-CTR 密钥，nonce 固定。虽然每一步都调用 `secrets.choice`，有限状态变换的输出分布仍然高度偏斜。

## 解题过程

把 $g\in\mathbb{Z}_{256}$ 看作 256 状态的马尔可夫链。对公开的四元集合枚举全部 $(a,b)$，构造一步转移概率；初始的 20 次求和也可直接卷积。迭代到分布收敛后，取概率最高的若干字节：

```python
dist = initial_distribution(sample)
for _ in range(enough_rounds):
    dist = affine_step(dist, sample)  # g -> a*g+b mod 256
candidates = sorted(range(256), key=dist.get, reverse=True)[:12]
```

服务一次最多返回 10 份 flag 密文。对每份密文枚举六字节密钥的高概率组合；$12^6=2,985,984$，在已知固定 nonce 和固定填充的条件下可以直接验证：

```python
for key6 in product(candidates, repeat=6):
    key = pad(bytes(key6), 16)
    pt = AES.new(key, AES.MODE_CTR, nonce=nonce).decrypt(ct)
    if pt.startswith(b"grey{") and pt.endswith(b"}"):
        print(pt.decode())
        break
```

恢复出：

```text
grey{G3T_Rand0m_Byte-is_Still_Bi@s_Oof_7nwh8eQfV5e8eZwC}
```

## 方法总结

安全随机源只能保证每次抽样本身不可预测，不能自动修复后续算法造成的统计偏差。这里仿射迭代发生在极小的 $\mathbb{Z}_{256}$ 上，且系数只来自四个值，稳态分布远非均匀。先建模输出分布、再按概率排序枚举，比盲目搜索全部 $2^{48}$ 个六字节密钥快得多。
